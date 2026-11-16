# Invoice Digitization with an LLM — Drive → Sheets (n8n, prototype)

PDF invoices dropped into a Google Drive folder are read, structured by an LLM against a JSON Schema, and appended to a Google Sheet. The interesting part is the **two-layer output parsing**: a structured parser wrapped in an **auto-fixing parser** that sends malformed output back to the model for correction.

## Architecture

```
Google Drive Trigger (fileCreated in folder)
   ─▶ Download file ─▶ Extract from File (PDF text)
   ─▶ Code: normalize text (single spaces, strip special characters)
   ─▶ Loop Over Items (one invoice at a time)
         ─▶ Basic LLM Chain (gpt-4o-mini)
               └─ Auto-fixing Output Parser
                     └─ Structured Output Parser (JSON Schema)
         ─▶ Append row in sheet ─▶ back to the loop
```

## Prerequisites

| Service | Purpose |
|---|---|
| Google Drive | Watched folder (trigger) and download |
| OpenAI | `gpt-4o-mini` for extraction and for the fixing pass |
| Google Sheets | Destination register |

## Step by step

### 1. Intake and cleaning
- **Trigger:** `fileCreated` in a specific folder (polling).
- **Extract from File:** PDF → text.
- **Code node:** joins lines, collapses whitespace and removes anything that isn't a letter, number, space, dot or comma. Cleaner input = fewer tokens and fewer parsing surprises.

### 2. Extraction prompt
The invoice text goes inside `<factura>` tags. Required fields: description (1–2 lines), type (Recurrente Mensual / Recurrente Anual / Pago Único), invoice date, payment date, invoice number, total price and currency (USD, EUR, GBP, ARS). Optional: purchase order, vendor, vendor tax ID, line items, prices with and without VAT, payment terms, bank details, late-payment penalties, delivery terms, contacts. *"If you can't find an item, leave it blank and move on."*

### 3. Output validation (two layers)
| Layer | What it does |
|---|---|
| **Structured Output Parser** | JSON Schema with `required` fields, date formats, an `enum` for the invoice type, and nested objects for vendor and client addresses |
| **Auto-fixing Output Parser** | If the output fails the schema, it sends the instructions, the bad completion and the error back to the model and asks for a corrected version |

### 4. Sheet mapping
`Datos remitente` ← vendor · `Cif` ← vendor tax ID · `Numero de factura` · `Fecha` · `Producto` · `Precio unitario` · `Cantidad` · `Importe` · `Subtotal` · `IVA` · `Total`.

## Testing
Drop two PDFs into the folder at once → two rows, one per invoice. This export includes the fix for duplicated rows when several files arrive together.

## Known limitations
- Scanned (image-only) PDFs produce no text; they need an OCR step first (see `../telegram-expense-tracker`).
- Line items are flattened into one row per invoice.

## Stack
n8n · OpenAI (gpt-4o-mini) · Google Drive · Google Sheets

## Status
Prototype. Exported inactive; folder and sheet IDs removed.
