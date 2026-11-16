# n8n AI Automation Portfolio — Gonzalo Insua

Sanitized n8n workflows (plus two earlier Make.com blueprints) from systems I built for my own brand, for clients, and for technical assessments. Credentials, API keys, webhook IDs, phone numbers, emails, client names and Google/Airtable resource IDs were removed or replaced with placeholders such as `<API_KEY>`. All n8n workflows are exported inactive.

## AI agents

| Project | What it is | Status |
|---|---|---|
| [`whatsapp-sales-agent`](./whatsapp-sales-agent) | Tool-calling WhatsApp agent with live Mercado Libre data: main agent, 4 tool sub-workflows, Redis message buffer, OAuth watchdog | Own D2C brand, no longer running |
| [`real-estate-ai-agent`](./real-estate-ai-agent) | Multi-agent assistant for a real estate agency: LLM router, sales agent (Airtable search + vector store), scheduling agent (Google Calendar) and 3 tool sub-workflows | Client project (with Tiki AI Solutions); ran in production, later retired by the client for cost |
| [`rag-sales-assistant-hitl`](./rag-sales-assistant-hitl) | Sales-reply assistant with retrieval over a vector store and human approval before sending | Prototype |

## Document and expense automation

| Project | What it is | Status |
|---|---|---|
| [`telegram-expense-tracker`](./telegram-expense-tracker) | Multimodal expense and invoice tracker over Telegram (text, photo, PDF, voice), with duplicate detection and review flows | Delivered and handed off to a client |
| [`expense-assistant-reports`](./expense-assistant-reports) | Variant of the tracker with Argentine tax breakdown, natural-language spending questions and a monthly HTML report by email | Delivered to a private client |
| [`expense-system-supabase`](./expense-system-supabase) | Three-workflow design on Supabase: email receipt intake, Telegram recorder, review reminders | Prototype |
| [`invoice-digitization-llm`](./invoice-digitization-llm) | Google Drive PDFs → text extraction → LLM chain with structured, auto-fixing output → Google Sheets | Prototype |
| [`make-blueprints`](./make-blueprints) | The original Make.com version of the expense tracker and an email-receipts-to-Sheets scenario | Delivered (later migrated to n8n) |

## Operations automation

| Project | What it is | Status |
|---|---|---|
| [`appointment-reminders-whatsapp`](./appointment-reminders-whatsapp) | Daily reminders for next-day appointments across 3 calendars, sent through Chatwoot with message variants and send delays | Prototype |

## Technical assessments (mock data)

| Project | What it is |
|---|---|
| [`slack-thread-summarizer`](./slack-thread-summarizer) | Webhook → clean and validate 100+ mock Slack threads → batched LLM summaries with structured output → post to Slack, with an error branch |
| [`lead-enrichment-scoring`](./lead-enrichment-scoring) | Fetch leads → enrich from an API → LLM scoring with structured output → save to CRM |

## Every project README covers
Overview · architecture diagram · prerequisites (services, credentials, tables) · step-by-step node configuration · prompts and output schemas · error handling · testing · known limitations.

## How to import
- **n8n:** Workflows → Import from file, then attach your own credentials and replace the placeholders.
- **Make.com:** Create a new scenario → Import blueprint.

## Design principles
- Deterministic logic first; LLMs only where the input is ambiguous.
- Validate LLM output before anything is written or sent.
- Human checkpoints wherever a mistake would cost something.
