# Daily Appointment Reminders — 3 Calendars → Chatwoot (n8n, prototype)

Every day the workflow reads the next day's appointments from three clinic calendars and sends each patient a reminder through Chatwoot. **No LLM:** the whole flow is deterministic, which keeps it cheap and predictable.

## Overview
- Reads three Google Calendars in parallel and merges them.
- Pulls the patient's phone from the event description (`Tel: …`).
- Finds the contact in Chatwoot or creates it, then finds the open conversation.
- Picks one of three message variants at random and waits a random 1–25 seconds before sending, so a batch of reminders doesn't look automated or hit rate limits.
- Logs every message sent to PostgreSQL.

## Architecture

```
Schedule (every 24h) ─▶ Calculate Date Range (tomorrow 00:00 → day after 00:00)
   ├─▶ Calendar – Clinic 1 ─┐
   ├─▶ Calendar – Clinic 2 ─┼─▶ Merge (append) ─▶ Filter (has a title)
   └─▶ Calendar – Clinic 3 ─┘
 ─▶ Loop Over Items
      ─▶ Extract Phone Number (regex on description)
      ─▶ Chatwoot: search contact ─▶ exists? ── no ─▶ create contact
      ─▶ Set Contact ID ─▶ Chatwoot: get conversations
      ─▶ Randomize (variant 1–3, delay 1–25s, first open conversation)
      ─▶ Variant 1 | 2 | 3 ─▶ Wait ─▶ Chatwoot: send message
      ─▶ PostgreSQL: insert into chat_history ─▶ next item
```

## Prerequisites

| Service | Purpose |
|---|---|
| Google Calendar | One calendar per clinic |
| Chatwoot | Contacts, conversations, outgoing messages (`CHATWOOT_ACCOUNT_ID` env var + API credential) |
| PostgreSQL | `chat_history(conversation_id, message_content, message_type, phone_number, created_at)` |

**Event convention:** title = patient name; description contains `Tel: +54 9 11 1234-5678`.

## Message variants
1. *"Hola {name}, confirmamos tu cita para mañana {time}…"*
2. *"Hola {name}, te recordamos tu consulta programada para mañana {time}…"*
3. *"Estimado/a {name}, este es un recordatorio de su cita médica mañana {time}…"* (formal)

## Testing
Create a test event for tomorrow with your own number in the description → run manually → check the Chatwoot conversation and the `chat_history` row.

## Known limitations
- The phone must follow the `Tel:` convention; there is no check for events without it, so they reach Chatwoot with an empty phone.
- The loop's batching wiring should be reviewed before scaling to large calendars.

## Stack
n8n · Google Calendar · Chatwoot API · PostgreSQL

## Status
Prototype. Exported inactive; calendar IDs and credentials removed.
