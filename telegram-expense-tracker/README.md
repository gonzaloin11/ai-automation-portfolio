# Telegram Expense & Invoice Tracker (n8n)

Built for a winery client and handed off with a user manual. The team sends invoices and receipts to a Telegram bot as **photos, PDFs, voice notes or plain text**; the bot extracts the data, blocks duplicates, stores everything in PostgreSQL and Google Sheets, and runs three approval flows from chat commands. Measured result: **55% less time on manual data entry** (timed manual entry vs. the bot).

Originally built in Make.com (see `../make-blueprints`) and migrated to n8n.

## Overview

- **One entry point, four input types:** PDF, image, voice note or free text, routed to the right extractor.
- **Extraction:** PDF text extraction, Mistral OCR for photos, OpenAI transcription for audio, then an OpenAI model returns structured JSON.
- **Validation before every write:** parse check, duplicate check in PostgreSQL, then insert.
- **Dual storage:** PostgreSQL as the source of truth, Google Sheets for the client's day-to-day view, original files in Google Drive.
- **Stateful chat flows** (`/revision`, `/mediopago`, `/comprobante`) with session state in PostgreSQL.
- **Error logging:** LLM, parse and database failures are written to an errors table and the user gets a clear message.

## Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│ ROUTER                                                                     │
│ Telegram Trigger ─▶ Verificar Sesión (PostgreSQL) ─▶ Router Principal      │
│   "cancelar" ─▶ reset session        /start ─▶ welcome                     │
│   /revision ─▶ Review flow           /mediopago ─▶ Payment-method flow     │
│   /comprobante ─▶ Payment-receipt flow                                     │
│   active session (revision | mediopago | comprobante) ─▶ that flow's step  │
│   anything else ─▶ EXPENSE INTAKE                                          │
└────────────────────────────────────────────────────────────────────────────┘
                                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ EXPENSE INTAKE                                                             │
│ Tipo de Mensaje                                                            │
│   PDF   ─▶ download ─▶ Drive upload + Extract PDF text ─┐                  │
│   Image ─▶ download ─▶ Drive upload ─▶ Mistral OCR ─────┤                  │
│   Audio ─▶ download ─▶ OpenAI transcription (es) ───────┼─▶ Consolidar     │
│   Text  ─▶ Preparar Texto ──────────────────────────────┘     Datos        │
│                                                                ▼           │
│ Extraer Datos (gpt-4.1-mini, temp 0.1) ─▶ Parsear JSON ─▶ Parse OK?        │
│      │ fail ─▶ log llm_failure          no ─▶ log parse_failure + tip      │
│      ▼                                                                     │
│ Verificar Duplicado ─▶ ¿Es Nuevo? ── no ─▶ "already registered"            │
│                           │ yes                                            │
│                           ▼                                                │
│ Insertar en DB ─▶ OK? ── no ─▶ log db_failure ─▶ "saved for manual review" │
│                     │ yes                                                  │
│                     ▼                                                      │
│ Guardar en Sheets (append or update) ─▶ Confirmar Registro (Telegram)      │
└────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

### External services

| Service | Purpose | Required |
|---|---|---|
| Telegram Bot API | Channel, file downloads | ✅ |
| OpenAI | Extraction model (`gpt-4.1-mini`) + audio transcription | ✅ |
| Mistral AI | OCR on receipt photos | ✅ |
| PostgreSQL | Records, sessions, error log | ✅ |
| Google Drive | Original files (invoice and payment receipt) | ✅ |
| Google Sheets | Client-facing register | ✅ |

### Database
```sql
CREATE TABLE acme_registros (
  id_transaccion          TEXT PRIMARY KEY,
  fecha_fc                TEXT,
  servicio_comercio       TEXT,
  concepto                TEXT,
  importe                 TEXT,
  categoria               TEXT,
  tipo_de_gasto           TEXT,
  estado                  TEXT,     -- Pendiente → Revisada → Por Pagar → Pagado
  forma_de_pago           TEXT,
  link_comprobante        TEXT,
  link_comprobante_pago   TEXT,
  fecha_ingreso_pago      TIMESTAMPTZ,
  fecha_creacion_registro TIMESTAMPTZ DEFAULT NOW(),
  updated_at              TIMESTAMPTZ
);

CREATE TABLE acme_sesiones (
  chat_id                BIGINT PRIMARY KEY,
  estado                 TEXT DEFAULT 'idle',   -- idle | revision | mediopago | comprobante
  facturas_pendientes    JSONB,
  facturas_seleccionadas JSONB,
  updated_at             TIMESTAMPTZ
);

CREATE TABLE acme_errors (
  id             SERIAL PRIMARY KEY,
  chat_id        BIGINT,
  user_name      TEXT,
  nodo_origen    TEXT,
  tipo_error     TEXT,       -- llm_failure | parse_failure | db_failure
  error_message  TEXT,
  tipo_input     TEXT,
  raw_input      TEXT,
  extracted_json JSONB,
  created_at     TIMESTAMPTZ DEFAULT NOW()
);
```

## Expense intake, step by step

### 1. Input routing (`Tipo de Mensaje`)
| Rule | Branch |
|---|---|
| `document.mime_type == application/pdf` | PDF: download → Drive (`yyyy-MM-dd_HHmmss_<name>`) and text extraction in parallel |
| `photo.length > 0` | Image: download largest size → Drive → Mistral OCR |
| `voice` exists | Audio: download → OpenAI transcription, language `es` |
| `text` exists | Text: used as-is |

**Consolidar Datos** merges whichever text was produced plus the Drive link.

### 2. Extraction prompt (OpenAI, temperature 0.1)
Returns only JSON with:

| Field | Rule |
|---|---|
| `fecha_factura` | DD/MM/YYYY |
| `servicio_comercio` | vendor name |
| `concepto` | what was bought |
| `importe` | Argentine format (1.250,50) |
| `id_transaccion` | invoice/order/reference number if present |
| `categoria` | one of 17 fixed categories (Vivienda, Alimentación, Transporte, Servicios, Impuestos…) |
| `tipo_gasto` | "Único" or "Recurrente" |

### 3. Validation
- **Parsear JSON** strips code fences and parses; on failure sets `parse_failed` and keeps the first 500 characters of the raw output. If the model found no reference number, it generates a timestamp ID.
- **Parse OK?** → otherwise log `parse_failure` and reply: *"Reenviarlo con mejor calidad o escribir: 'Uber $2500 transporte'"*.
- **Verificar Duplicado** → `SELECT … WHERE id_transaccion = …`; duplicates get *"Esta transacción ya fue ingresada"* and nothing is written.
- **Insert OK?** → otherwise log `db_failure` with the extracted JSON and tell the user it's saved for manual review.

### 4. Storage and confirmation
Insert with `estado = 'Pendiente'` → Google Sheets *append or update* keyed on the transaction ID → Telegram summary (date, vendor, concept, amount, category, ID).

## Chat command flows

All three use the same pattern: the command starts a session in `acme_sesiones`, the bot sends a numbered list, and the user's next message is routed back to that flow by the session state.

| Command | Lists | User replies with | Effect |
|---|---|---|---|
| `/revision` | Up to 20 records in `Pendiente` | numbers (`1,3,5`), `todos` or `cancelar` | `estado = 'Revisada'` in DB and Sheets; the confirmation names who approved |
| `/mediopago` | Reviewed records with no payment method | numbers + method (efectivo, transferencia, débito, crédito, cheque…) | `forma_de_pago` set, `estado = 'Por Pagar'` |
| `/comprobante` | `Por Pagar` records with no payment receipt | a number, then a photo or PDF | file to Drive, `link_comprobante_pago`, `estado = 'Pagado'` |

`cancelar` at any point resets the session to `idle`.

## Error handling summary

| Failure | Logged as | User sees |
|---|---|---|
| OpenAI call fails | `llm_failure` | "Couldn't process it, try again or send it as text" |
| Model output isn't valid JSON | `parse_failure` | "Resend with better quality or type it" |
| Duplicate transaction | — | "Already registered" + ID |
| Insert fails | `db_failure` + extracted JSON | "Processed, saved for manual review" |
| Invalid selection in a flow | — | Explains the expected format |

## Testing
1. `/start` → welcome message.
2. Send a receipt photo → confirmation with the extracted fields; send it again → duplicate warning.
3. Type `Uber $2500 transporte` → registered from text.
4. `/revision` → reply `1` → record moves to *Revisada* in DB and Sheets.

## Known limitations
- SQL queries are built with string interpolation; production hardening should move them to parameterized queries.
- The duplicate check depends on the model finding a reference number; when it doesn't, a timestamp ID is generated, so the same receipt sent twice without a visible number isn't caught.

## Stack
n8n · Telegram Bot API · OpenAI (gpt-4.1-mini + transcription) · Mistral OCR · PostgreSQL · Google Sheets · Google Drive

## Status
Delivered to a client with a user manual and handed off. Exported inactive; client name, IDs and credentials removed.
