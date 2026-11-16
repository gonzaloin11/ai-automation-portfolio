# Real Estate AI Agent — Multi-Agent Assistant (n8n)

Multi-agent assistant for a real estate agency, co-built with Tiki AI Solutions (I built the agents and the integrations). It answers property questions, searches listings, books viewings in Google Calendar and confirms them by email. It was delivered and run in production; the client later retired it because of running costs.

## Overview

- **Intent router:** an LLM classifies every message as SALES, SCHEDULING or UNCLEAR and hands it to a specialist agent.
- **Sales agent:** searches listings in Airtable (generates the filter formula itself) and answers detail questions from a Supabase vector store of property descriptions.
- **Scheduling agent:** checks Google Calendar availability, collects the required data, creates the event and emails a confirmation.
- **Conversation agent:** handles greetings and messages with no property context.
- **Shared memory:** every agent reads the same PostgreSQL chat history, so the scheduling agent knows which property the sales agent discussed.
- **Companion tool sub-workflows:** schedule-and-qualify into the CRM (Tokko Broker), send the right brochure, hand off to a human.

## Files

| File | Role |
|---|---|
| `real-estate-agent-main.json` | Telegram channel version: router + 3 agents |
| `tool-schedule-viewing.json` | Sub-workflow: summarize the conversation, rate the lead 1–5, create the contact in Tokko Broker |
| `tool-send-brochure.json` | Sub-workflow: pick the development's brochure and send it by email |
| `tool-handoff-to-human.json` | Sub-workflow: notify the team (email + Slack) and tell the client a person will follow up |

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│ PHASE 1 — INTAKE                                                         │
│  Telegram Trigger ─▶ Extract User Data (sessionId, chat_id, first_name)  │
└──────────────────────────────────────────────────────────────────────────┘
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ PHASE 2 — ROUTING                                                        │
│  Router Classifier (gpt-4o, temp 0, + chat memory) ─▶ Route Switch       │
│      SALES ─────────┐   SCHEDULING ──────┐   UNCLEAR ───────┐            │
└─────────────────────┼────────────────────┼──────────────────┼────────────┘
                      ▼                    ▼                  ▼
┌──────────────────────────┐ ┌──────────────────────────┐ ┌─────────────────┐
│ SALES AGENT              │ │ SCHEDULING AGENT         │ │ CONVERSATION    │
│ gpt-4o, temp 0.7         │ │ gpt-4o, temp 0.3         │ │ AGENT           │
│ tools:                   │ │ tools:                   │ │ gpt-4.1-mini    │
│  • Airtable search       │ │  • Calendar – check      │ │ temp 0.5        │
│  • property_knowledge    │ │  • Calendar – create     │ │                 │
│    (Supabase vectors,    │ │  • Gmail – confirm visit │ │                 │
│    OpenAI embeddings)    │ │                          │ │                 │
└────────────┬─────────────┘ └────────────┬─────────────┘ └────────┬────────┘
             └──────────────┬─────────────┘                        │
                            ▼                                      ▼
              Prepare Response ─▶ Send to Telegram        Send a text message
         (all agents share PostgreSQL chat memory keyed by Telegram user id)
```

## Prerequisites

### External services

| Service | Purpose | Required |
|---|---|---|
| Telegram Bot API | Channel (trigger + replies) | ✅ |
| OpenAI | Chat models (router, agents) + embeddings (`text-embedding-3-large`, 1536 dims) | ✅ |
| Airtable | Listings table (price, city, zone, type, rooms…) | ✅ |
| Supabase (pgvector) | `property_embeddings` table with detailed property descriptions | ✅ |
| PostgreSQL | Chat memory shared by all agents | ✅ |
| Google Calendar | Availability and visit events | ✅ |
| Gmail | Visit confirmation, brochures, hand-off alerts | ✅ |
| Tokko Broker | Real estate CRM (schedule-viewing sub-workflow) | ⚪ |
| Slack | Hand-off alerts (via incoming webhook) | ⚪ |

### Credentials / environment
OpenAI · Telegram · Airtable personal access token · Supabase · PostgreSQL · Google Calendar OAuth2 · Gmail OAuth2 · Tokko header auth.
Sub-workflows read `ADMIN_EMAIL`, `SLACK_WEBHOOK_URL` and `BROCHURE_URL_*` from environment variables.

## Phase 1 — Intake
**Telegram Trigger** (`message` updates) → **Extract User Data**:

| Field | Value |
|---|---|
| `sessionId` | Telegram `from.id` (memory key for every agent) |
| `user_message` | message text |
| `chat_id` | chat id for the reply |
| `first_name` | `from.first_name` or "Usuario" |

## Phase 2 — Routing
**Router Classifier** — agent with `gpt-4o`, temperature 0, plus chat memory so follow-ups like "tenés más info?" are understood in context. It must answer one word:
- **SALES** — anything about properties: search, price, zone, features, "más detalles", "me interesa".
- **SCHEDULING** — clear intent to book or ask for viewing availability.
- **UNCLEAR** — greetings or messages with no property context.

The rule in the prompt: *if the message mentions properties in any way, it's SALES.* **Route Switch** sends the fallback to SALES too.

## Phase 3 — Specialist agents

### 3.1 Sales agent (`gpt-4o`, temperature 0.7)
Two tools, with explicit rules on when to use each:

| Tool | When | How |
|---|---|---|
| **Search records in Airtable** | First search: location, price, type, rooms | The model writes the `filterByFormula` itself (`$fromAI`); the prompt documents the exact field names and `{Field}` syntax |
| **property_knowledge** (Supabase vector store, retrieve-as-tool) | Follow-ups about one property: services, access, neighborhood, documents | Semantic search over `property_embeddings` with OpenAI `text-embedding-3-large` |

Rules: show at most 3 properties per search; suggest widening criteria when Airtable returns nothing; say so when the knowledge base has no extra detail.

### 3.2 Scheduling agent (`gpt-4o`, temperature 0.3)
Receives the client's name, session id, message and the current Buenos Aires date and time, plus the shared memory (so it can tell which property was discussed).

| Tool | Rule |
|---|---|
| **Google Calendar – Check** | Always first; checks the next 3–5 business days. Free slots = available |
| **Google Calendar – Create** | Only when name, email, phone, property name, address and time are all confirmed. Title `Visita - <property>`, client added as attendee, structured description |
| **Gmail – Confirmar visita** | Right after the event is created; HTML confirmation with property, date and address |

Visit slots in the prompt: Monday to Saturday at 10:00, 12:00, 16:00 and 18:00, one hour each.

### 3.3 Conversation agent (`gpt-4.1-mini`, temperature 0.5)
Greets, explains what the assistant can do and steers the user back to properties.

## Companion sub-workflows (Execute Workflow)

**`tool-schedule-viewing`** — input: name, email, phone, development, conversation.
1. *Resumir Conversación* — `gpt-4o-mini`, temp 0, structured output `{ "resumen": "..." }`.
2. *Calificar Lead* — `gpt-4o-mini`, temp 0, structured output `{ "calificacion": 1-5, "justificacion": "..." }` with a written rubric (5 = budget aligned + under 30 days + full contact data … 1 = no real intent).
3. *Tokko Broker CRM* — `POST /api/v1/web_contact/` with the summary, rating and tags `[development, "AI Agent", "Rating: N"]`.
4. Returns a confirmation sentence for the agent to relay.

**`tool-send-brochure`** — **Switch** on the development name → download that brochure PDF (`BROCHURE_URL_*`) → Gmail with the PDF attached → confirmation text. Unknown development → asks which one.

**`tool-handoff-to-human`** — in parallel: HTML email to `ADMIN_EMAIL` and a Slack Block Kit message → returns *"Ya notifiqué a un asesor humano…"*.

## Testing
- Use the built-in **"When chat message received"** trigger in the n8n editor to chat without Telegram.
- Flow to try: "busco una casa en el centro hasta 120 mil" → "contame más de la segunda" → "quiero visitarla el jueves" → provide name, email, phone.
- Check: the Calendar event is created only after all six data points are confirmed, and the confirmation email arrives.

## Known limitations (honest notes on this export)
- The three tool sub-workflows were built for the WhatsApp/CRM version and are **not wired** as tools in this Telegram export.
- A Redis message-buffer branch is included but **not connected** to the trigger in this version.
- The Airtable filter formula is generated by the model; a malformed formula returns no results instead of an error.

## Stack
n8n · OpenAI (gpt-4o, gpt-4.1-mini, gpt-4o-mini, embeddings) · Supabase vector store · Airtable · PostgreSQL · Google Calendar · Gmail · Telegram · Tokko Broker · Slack

## Status
Client project, delivered and run in production, later retired by the client because of running costs. Exported inactive; credentials, IDs and client names removed.
