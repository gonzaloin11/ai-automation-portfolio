# WhatsApp Sales Agent ("Lía") — n8n + Mercado Libre API

Customer-service and sales agent for my own collagen brand on WhatsApp. It answers product, price, stock and detail questions with **live Mercado Libre data** through tool calling, and cut response times from hours or days to under 3 minutes.

## Overview

- **Security filter** that drops the bot's own messages, group chats and status broadcasts (no reply loops).
- **Multimodal input:** text and voice notes (voice is transcribed before reaching the agent); any other media gets a polite "text or audio only" reply.
- **Redis anti-fragmentation buffer:** customers who type in several bursts get one answer, not five.
- **Tool-calling agent** with PostgreSQL chat memory (last 10 messages per customer).
- **Four Mercado Libre tools**, each a separate sub-workflow: search, price (with active promotions), stock and product detail.
- **Graceful failures:** friendly fallback messages if audio can't be read or the agent fails.
- **OAuth watchdog:** a separate scheduled workflow keeps the Mercado Libre token alive and emails an alert if the refresh fails.

## Files

| File | Role |
|---|---|
| `lia-main-agent.json` | Main workflow: webhook → filter → buffer → agent → reply |
| `tool-search-products.json` | Tool: list active products (optional keyword), real prices including promos |
| `tool-product-price.json` | Tool: current price, original price and discount for one item |
| `tool-product-stock.json` | Tool: stock summary for all active items, with low-stock and out-of-stock alerts |
| `tool-product-detail.json` | Tool: full detail for one item (description, attributes, shipping, link) |
| `watchdog-oauth-refresh.json` | Scheduled job: refreshes the Mercado Libre OAuth token every 2 hours |

## Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│ PHASE 1 — INGEST & GUARDRAILS                                              │
│  Webhook (Evolution API) ─▶ Security Filter ─▶ Route Input Type            │
│                               │ fromMe / group /     ├─ audio ─▶ Download  │
│                               │ broadcast → Discard  │          ─▶ Base64→ │
│                               ▼                      │          file ─▶    │
│                                                      │          Transcribe │
│                                                      ├─ text / extended    │
│                                                      └─ other ─▶ "text or  │
│                                                                 audio only"│
└────────────────────────────────────────────────────────────────────────────┘
                                     ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ PHASE 2 — REDIS BUFFER (anti-fragmentation)                                │
│  Push message to list ─▶ Get list ─▶ Switch:                               │
│     • newer message arrived  → stop (a later execution will answer)        │
│     • quiet for 7s           → delete list ─▶ join messages                │
│     • otherwise              → Wait 7s ─▶ check again                      │
└────────────────────────────────────────────────────────────────────────────┘
                                     ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ PHASE 3 — AGENT                                                            │
│  AI Agent (OpenAI chat model) + PostgreSQL memory (10 msgs)                │
│     tools: search · price · stock · detail  (Execute Workflow)             │
│  Agent Error? ── ok ─▶ Send WhatsApp reply                                 │
│               └─ fail ─▶ "Try again in a few minutes"                      │
└────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────── separate workflow ────────────────────────────┐
│ WATCHDOG — every 2h: read token expiry (PostgreSQL)                        │
│   < 2h left? ─▶ POST /oauth/token (refresh) ─▶ OK? ─▶ UPDATE ml_tokens     │
│                                               └─ no ─▶ Gmail alert         │
└────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

### External services

| Service | Purpose | Required |
|---|---|---|
| Evolution API | WhatsApp connection (webhook in, `sendText` out, media download) | ✅ |
| OpenAI | Chat model for the agent + audio transcription | ✅ |
| Redis | Message buffer per chat | ✅ |
| PostgreSQL | Chat memory + Mercado Libre token storage | ✅ |
| Mercado Libre API | Items, descriptions, seller promotions, OAuth | ✅ |
| Gmail | Watchdog alert | ⚪ Recommended |

### Credentials in n8n
- OpenAI API key · Redis · PostgreSQL · Gmail OAuth2.
- Evolution API values (`server_url`, `instance`, `apikey`) arrive in the webhook payload and are reused for replies.
- Mercado Libre app `client_id` / `client_secret` go in the watchdog's refresh request (placeholders `<ML_CLIENT_ID>`, `<ML_CLIENT_SECRET>`). Store them as n8n credentials or environment variables, never inline.

### Database
```sql
-- Chat memory (created by n8n's Postgres Chat Memory node)
-- table: vitagel_chat_histories, session key: "vitagel_<remoteJid>"

-- OAuth token store read by every tool and refreshed by the watchdog
CREATE TABLE ml_tokens (
  app_id        TEXT PRIMARY KEY,      -- 'vitagel'
  access_token  TEXT,
  refresh_token TEXT,
  token_type    TEXT,
  expires_in    INTEGER,
  scope         TEXT,
  user_id       BIGINT,
  expires_at    TIMESTAMPTZ,
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);
```

## Phase 1 — Ingest and guardrails

**Webhook** — `POST /webhook/vitagel-lia-wapp`, called by Evolution API on every incoming message.

**Security Filter (IF)** — continues only when all are true:
- `data.key.fromMe == false` → never answers its own messages (anti-loop).
- `remoteJid` does not contain `@g.us` → ignores groups.
- `remoteJid != status@broadcast` → ignores status updates.

**Route Input Type (Switch)** — by message type:
- **Audio** → `POST {server_url}/chat/getBase64FromMediaMessage/{instance}` → Code node converts base64 to a binary file → OpenAI transcription. `Audio Error?` sends *"No pude escuchar tu audio 😕 ¿Podés escribirme lo que necesitás?"* if nothing was transcribed.
- **Text / extended text** → normalized into `chat_id`, `instance_name`, `user_message`.
- **Fallback (images, stickers…)** → *"Por ahora solo puedo leer mensajes de texto y audios 🙂"*.

**Extract User Data** — builds `sessionId = "vitagel_<remoteJid>"` (one memory thread per customer).

## Phase 2 — Redis buffer

| Step | Node | Config |
|---|---|---|
| 2.1 | Redis push | list `vitagel:buffer:{chat_id}`, value `{message, sessionId, date_time}`, push to tail |
| 2.2 | Redis get | read the whole list |
| 2.3 | Switch Buffer | *ignore* if the last message in the list isn't this execution's message (a newer one will answer) · *continue* if the last message is older than now − 7s · *wait* otherwise |
| 2.4 | Wait | 7 seconds, then back to 2.2 |
| 2.5 | Redis delete + Set | clear the list and join all messages with `\n` into `concat_message` |

Result: three quick messages like "hola" / "precio" / "del de 30 días?" become one question.

## Phase 3 — Agent

**AI Agent** — input `concat_message`. The system prompt defines Lía as a consultative sales advisor:
- Introduces herself in the first message and asks for the customer's name.
- Argentine Spanish, short plain-text replies (no markdown, 1–2 emojis max).
- Asks what the customer needs **before** giving a price, then uses the tools for real prices and stock.
- **Link rule:** never sends a link unless the customer explicitly asks; then sends the product permalink returned by the detail tool.

| Setting | Value |
|---|---|
| Model | OpenAI chat model (`gpt-5-mini` in this export), temperature 1 |
| Memory | PostgreSQL Chat Memory, table `vitagel_chat_histories`, window 10 |
| Tools | 4 × Execute Workflow tools (below) |
| Error handling | `continueOnFail` on the agent and the audio chain |

**Agent Error?** → reply through `POST {server_url}/message/sendText/{instance}`, or the fallback *"Estoy teniendo un pequeño problema, intentá de nuevo en unos minutos 🙏"*.

### Tools (sub-workflows)

All four start with `SELECT access_token FROM ml_tokens WHERE app_id = 'vitagel'` and call the API with `Authorization: Bearer <token>`.

| Tool | Input | API calls | What the Code node returns |
|---|---|---|---|
| **Buscar Productos** | `query` (optional) | `GET /users/{seller}/items/search?status=active&limit=50` → `GET /items?ids=…` (batch) → `GET /seller-promotions/items/{id}` per item | Title, real price (promo applied when active), stock, duration in days, link; excludes non-sellable IDs |
| **Precio Producto** | `item_id` | `GET /items/{id}` → `GET /seller-promotions/items/{id}` | Final price, original price, discount %, price source (promo vs list) |
| **Stock Productos** | — | items search → batch `GET /items?ids=…` | Totals, out-of-stock list, low stock (≤ 5 units), per-item detail |
| **Detalle Producto** | `item_id` | `GET /items/{id}` → `GET /items/{id}/description` | Price and discount, stock, brand/units/weight/flavor attributes, shipping, description, link |

The tool descriptions tell the model *when* to call each one (e.g. "use Precio when the customer asks specifically about price").

## Watchdog (separate workflow)

1. **Schedule** every 2 hours.
2. **PostgreSQL** reads `expires_at` and computes `hours_remaining`.
3. **IF** `hours_remaining < 2` → `POST https://api.mercadolibre.com/oauth/token` with `grant_type=refresh_token`.
4. **Refresh OK?** (non-empty `access_token`) → `UPDATE ml_tokens …, expires_at = NOW() + expires_in seconds`.
5. Otherwise → **Gmail alert** with timestamp, remaining hours and the error.

It runs independently of the main OAuth manager, so a silent failure there doesn't take the bot down.

## Error handling summary

| Failure | Behavior |
|---|---|
| Message from the bot, a group or a broadcast | Discarded |
| Unsupported media | Polite "text or audio" reply |
| Audio download/transcription fails | "Couldn't hear your audio" reply |
| Agent/model fails | "Try again in a few minutes" reply |
| Mercado Libre item not found | Tool returns a readable error string to the agent |
| Token refresh fails | Email alert |

## Testing

```bash
# Simulate an Evolution API text message
curl -X POST https://<your-n8n-host>/webhook/vitagel-lia-wapp \
  -H "Content-Type: application/json" \
  -d '{"instance":"<instance>","server_url":"https://<your-evolution-api-host>","apikey":"<EVOLUTION_API_KEY>",
       "data":{"key":{"remoteJid":"5490000000000@s.whatsapp.net","fromMe":false,"id":"TEST1"},
               "message":{"conversation":"hola, ¿qué precio tiene el de 30 días?"}}}'
```
- Send three messages within 7 seconds → expect **one** reply.
- Reset a test conversation: `DELETE FROM vitagel_chat_histories WHERE session_id = 'vitagel_<remoteJid>';`

## Known limitations
- Product durations and excluded item IDs are hard-coded in the search tool's Code node; a catalog change needs a code edit.
- The seller ID is fixed in the search/stock URLs (`<SELLER_ID>`).
- No rate limiting per customer.

## Stack
Evolution API (WhatsApp) · n8n · OpenAI (chat + transcription) · Redis · PostgreSQL · Mercado Libre API · Gmail

## Status
Built for my own D2C brand; no longer running. Exported inactive, credentials and IDs removed.
