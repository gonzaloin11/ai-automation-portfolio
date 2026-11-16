# Expense Assistant with AI Queries and Monthly Reports (n8n)

A variant of the Telegram expense tracker, built for a private client. It adds **Argentine tax breakdown** (net, VAT, other taxes), **natural-language questions about spending** routed to an AI sub-workflow, and a **monthly HTML report** sent by email.

## Overview

- Same four inputs as the base tracker: PDF, photo (Mistral OCR), voice note (transcription) and text.
- **Extraction tuned for Argentine invoices:** invoice type (Factura A/B/C, credit note, receipt, ticket), net amount, VAT, other taxes and their detail, total.
- **Strict category list** (15 lowercase categories, no accents) so the report can group reliably.
- **Questions vs. expenses:** a regex decides whether a text message is a question ("¿cuánto gasté en combustible?") or an expense to record. Questions go to an AI sub-workflow and the answer comes back on Telegram.
- **Monthly report:** on the 1st of each month at 09:00, or on demand with `/reporte`, it reads the register, aggregates it and emails an HTML report.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Telegram Trigger ─▶ /start? ── yes ─▶ welcome                           │
│                        │ no                                             │
│                        ▼                                                │
│                    /reporte? ── yes ─▶ REPORT                           │
│                        │ no                                             │
│                        ▼                                                │
│ Obtener Último ID ─▶ Tipo de Mensaje                                    │
│   PDF ─▶ Drive + PDF text ───────────────┐                              │
│   Photo ─▶ Drive ─▶ Mistral OCR ─────────┤                              │
│   Voice ─▶ OpenAI transcription ─────────┼─▶ Consolidar ─▶ EXTRACTION   │
│   Text ─▶ ¿Es Consulta IA? (regex)       │                              │
│              ├─ question ─▶ Execute Workflow "Consulta IA" ─▶ reply     │
│              └─ expense ─────────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘
 EXTRACTION: gpt-4.1-mini (temp 0.1) ─▶ Parsear JSON (safe defaults)
             ─▶ duplicate check (PostgreSQL) ─▶ insert ─▶ Sheets ─▶ confirm

 REPORT: Schedule "0 9 1 * *"  or  /reporte
         ─▶ Leer Gastos (Sheets) ─▶ Procesar Datos (totals by category & month)
         ─▶ Generar HTML ─▶ Gmail ─▶ Telegram summary
```

## Prerequisites

| Service | Purpose |
|---|---|
| Telegram Bot API | Channel |
| OpenAI | Extraction (`gpt-4.1-mini`) + transcription |
| Mistral AI | OCR on photos |
| PostgreSQL | `client.registros` table, duplicate check |
| Google Drive / Sheets | Files and the register the report reads |
| Gmail | Monthly report |
| Second n8n workflow | "Consulta IA" agent that answers spending questions (called by ID, not included) |

## Extraction prompt (fields)

| Field | Rule |
|---|---|
| `fecha` | DD/MM/YYYY |
| `nombre_empresa` | issuer |
| `concepto` | main item |
| `tipo_comprobante` | Factura A, Factura B, Factura C, Nota de Crédito, Recibo, Ticket, Otro |
| `importe_sin_impuestos`, `iva`, `otros_impuestos` | numbers only, no currency symbol |
| `otros_impuestos_detalle` | e.g. "Percepción IVA", "Percepción IIBB" |
| `total` | number |
| `id_transaccion` | required; timestamp if the document has none |
| `categoria` | exactly one of: combustible, alimentacion, transporte, servicios, alquiler, impuestos, seguros, mantenimiento, salud, educacion, entretenimiento, vestimenta, tecnologia, hogar, otros |
| `tipo_gasto` | "Único" or "Recurrente" |

If the model output doesn't parse, **Parsear JSON** stores a safe placeholder (vendor "No identificado", concept "Pendiente revisión", category "otros", total 0) so nothing is lost and the record is easy to find and fix.

## Question routing
`¿Es Consulta IA?` runs a case-insensitive regex on the text (cuánto, cuál, qué, total, suma, gasté, pendiente, factura, detalle, mostrar, resumen, `?`…). Match → **Execute Workflow** (the agent workflow) → its `respuesta` is sent back on Telegram. No match → recorded as an expense.

## Monthly report
1. **Leer Gastos** reads the whole register from Google Sheets.
2. **Procesar Datos** normalizes categories (lowercase, accents stripped), parses totals, and aggregates **by category** and **by month** (Spanish month names), plus the transaction count and the top category.
3. **Generar HTML** builds an inline-styled email: header totals, category table with icons, monthly table and a detail table.
4. **Gmail** sends it; Telegram confirms with total, number of transactions and the biggest category.

## Testing
- Send a Factura A as PDF → check net, VAT and total are split.
- Type `¿cuánto gasté en combustible este mes?` → answered by the agent, not recorded.
- Type `/reporte` → email arrives and Telegram shows the summary.

## Known limitations
- The question/expense split is a regex, not a classifier; short confirmations like "ok" or "dale" are treated as questions.
- The answering agent lives in a separate workflow that isn't part of this export.

## Stack
n8n · Telegram · OpenAI · Mistral OCR · PostgreSQL · Google Sheets · Google Drive · Gmail

## Status
Delivered to a private client. Exported inactive; names, IDs and credentials removed.
