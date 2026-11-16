# Make.com Blueprints — Original Expense Automations

The first versions of the expense system, built in Make.com and delivered to a client before I migrated them to n8n (`../telegram-expense-tracker`). Kept here to show the same problem solved on two platforms.

## `telegram-expense-tracker.blueprint.json`

A large router-based scenario (123 modules) for a Telegram expense bot.

```
Telegram Watch Updates ─▶ Data store: last transaction ID
  ├─ Message routes
  │    ├─ PDF ─▶ PDF.co → images ─▶ OpenAI vision (gpt-4o-mini) ─▶ parse ─▶ record
  │    ├─ Payment receipt ─▶ Drive upload ─▶ update data store + Sheets row
  │    ├─ Text ─▶ OpenAI structured data ─▶ data store ─▶ Sheets + Google Doc ─▶ reply
  │    ├─ Audio ─▶ Whisper ─▶ OpenAI structured data ─▶ duplicate check ─▶ record | "already registered"
  │    └─ Commands ─▶ calls other scenarios by webhook (pending reviews, receipt requests)
  └─ Callback routes (inline buttons)
       ├─ Payment method ─▶ update Sheets with method and date ─▶ "requires receipt?" branch
       ├─ Mark as seen ─▶ update data store + Sheets
       ├─ Ask for payment method ─▶ keyboard
       └─ Ask for receipt ─▶ wait for file ─▶ link it to the invoice
```

**Patterns used:** Make data stores as the transaction index and session state; routers with filters for every message type and callback; separate scenarios called by webhook for the review and receipt flows; OpenAI "transform text to structured data" to read IDs out of callback payloads.


## `email-receipts-to-sheets.blueprint.json`

```
Custom mail hook ─▶ Drive upload ─▶ OpenAI vision (gpt-4o-mini)
  ├─ result ─▶ parse JSON ─▶ Sheets row ─▶ OpenAI assistant step ─▶ update row
  │            ─▶ data store duplicate check
  │                 ├─ new ─▶ record ID + email "Carga Factura"
  │                 └─ dup ─▶ email "Carga Factura Duplicada"
  └─ no result ─▶ email back asking for a better image
```

## Import
Make.com → Create a new scenario → ⋯ → Import Blueprint, then reconnect Telegram, OpenAI, Google, PDF.co and recreate the data stores. Webhook URLs, sheet IDs and connections were removed.

## Status
Delivered to a client and later replaced by the n8n version.
