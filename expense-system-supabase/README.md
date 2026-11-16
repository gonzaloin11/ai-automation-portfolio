# Expense System on Supabase — Three-Workflow Design (n8n, prototype)

A redesign of the expense tracker as three cooperating workflows over one Supabase table: **email intake**, a **Telegram recorder with inline buttons**, and **scheduled reminders**. Built as a prototype to test a design with inline-keyboard actions instead of typed commands.

## Overview

- **Workflow A — Email receipts:** reads an inbox over IMAP, keeps only image/PDF attachments, converts PDFs to images, extracts the data with an OpenAI vision model, deduplicates and records it, then emails back a confirmation or a duplicate notice.
- **Workflow B — Telegram recorder:** registers expenses from photos, PDFs, voice notes and text; handles inline-button callbacks to set the payment method or mark a record as seen.
- **Workflow C — Reminders:** every 6 hours (or on a webhook call) sends one Telegram message per pending record, each with buttons to act on it.

## Architecture

```
┌─────────────────────────── A · EMAIL INTAKE ───────────────────────────┐
│ IMAP (unread, last 24h, every minute) ─▶ Has attachments? ─▶ Split    │
│  ─▶ Is image/PDF? ─▶ Drive upload ─▶ Drive download ─▶ Is PDF?         │
│        PDF ─▶ PDF.co → JPG (page 1) ─▶ download ─┐                     │
│        image ────────────────────────────────────┴▶ OpenAI vision      │
│  ─▶ Parse JSON & generate ID ─▶ duplicate? (Supabase)                  │
│        new ─▶ insert ─▶ Sheets ─▶ Gmail "Gasto registrado"             │
│        dup ─▶ Gmail "Gasto duplicado"                                  │
└────────────────────────────────────────────────────────────────────────┘
┌───────────────────────── B · TELEGRAM RECORDER ────────────────────────┐
│ Trigger (message + callback_query) ─▶ Route Message Type               │
│   callback ─▶ parse "metodo:|visto:|pago:" ─▶ update Supabase ─▶ ack   │
│   photo / PDF ─▶ Drive ─▶ OpenAI vision ─▶ parse ─▶ insert ─▶ confirm  │
│   voice ─▶ transcription ─▶ OpenAI structure ─▶ parse ─▶ insert        │
│   text ─▶ OpenAI structure ─▶ valid? ─▶ insert | "send comercio monto" │
│   /start ─▶ welcome                                                    │
└────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────── C · REMINDERS ─────────────────────────────┐
│ Schedule (6h) or Webhook ─▶ active chat                                │
│   ├─ unseen records ─▶ loop: message + [Marcar como visto]            │
│   └─ no payment method ─▶ loop: message + [Transferencia|Cheque|…]    │
│ ─▶ Webhook response {"success": true}                                  │
└────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

| Service | Purpose |
|---|---|
| Supabase | `expense_records` and `chat_state` tables |
| OpenAI | `gpt-4o-mini` (vision and text), transcription |
| PDF.co | PDF → JPG conversion (header auth) |
| Gmail / IMAP | Email intake and replies |
| Google Drive / Sheets | Files and register |
| Telegram Bot API | Bot, inline keyboards, callbacks |

```sql
CREATE TABLE expense_records (
  id_transaccion     TEXT PRIMARY KEY,
  servicio_comercio  TEXT,
  concepto           TEXT,
  importe            TEXT,
  categoria          TEXT,
  tipo_gasto         TEXT,
  forma_pago         TEXT,
  vista_por_usuario  BOOLEAN DEFAULT FALSE,
  link_comprobante   TEXT,
  fecha_creacion     TIMESTAMPTZ DEFAULT NOW()
);
CREATE TABLE chat_state ( chat_id BIGINT PRIMARY KEY );
```

## Workflow A — details

| Step | Config |
|---|---|
| Trigger | IMAP, every minute, unread mail received in the last 24h, download attachments |
| Filter | attachment MIME matches `image/` or `application/pdf`; other files end the branch |
| PDF.co | `POST /v1/pdf/convert/to/jpg`, page 1, synchronous |
| Extraction | `gpt-4o-mini`, temp 0.3, returns `FECHA DEL COBRO`, `SERVICIO / COMERCIO`, `CONCEPTO`, `IMPORTE`, `ID DE TRANSACCION`, `CATEGORIA` (9 fixed groups), `TIPO DE GASTO`, `ORIGEN DEL COBRO` |
| ID | if the receipt has no ID, it's built from the date (YYYYMMDD) and the amount, so re-sending the same receipt is still caught as a duplicate |
| Reply | to the original sender: "Gasto Registrado: <vendor>" or "Gasto Duplicado Detectado" |

## Workflow B — callbacks

Inline buttons carry their action in `callback_data`:

| Prefix | Action | Supabase update |
|---|---|---|
| `metodo:<method>\|<id>` | set payment method | `forma_pago = <method>` |
| `visto:<id>` | mark as seen | `vista_por_usuario = true` |
| `pago:<id>` | ask for a method | sends the keyboard: Transferencia · Cheque · Efectivo · Tarjeta |

Text messages that the model can't turn into a valid expense get: *"Envía imagen, audio o texto con formato: comercio monto concepto"*.

## Workflow C — reminders
- Up to 10 unseen records and 10 records without a payment method, newest first.
- One message per record (batch size 1) with its own buttons, so each record can be closed in one tap.
- "No pending" messages when the lists are empty; the webhook caller receives `{"success": true, "message": "Reminders processed"}`.

## Testing
1. Email a receipt PDF to the watched inbox → confirmation email; send it again → duplicate email.
2. Send a photo to the bot → "Gasto registrado".
3. Call `POST /webhook/expense-reminders` → reminders with buttons; tap *Transferencia* → record updated.

## Known limitations
- Prototype: the reminders go to a single active chat (`LIMIT 1`).
- The Google Sheets document ID is left empty in this export.

## Stack
n8n · Supabase · OpenAI (gpt-4o-mini vision + transcription) · PDF.co · Gmail/IMAP · Google Drive · Google Sheets · Telegram

## Status
Prototype. Exported inactive; credentials removed.
