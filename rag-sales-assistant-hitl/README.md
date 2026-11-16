# RAG Sales Assistant with Human Approval (n8n, prototype)

Drafts the first sales reply to an inbound real estate lead using a vector store as context, and **never sends it without a human OK**. If the reviewer asks for changes, a second agent rewrites only what was asked and the draft goes back for approval.

## Architecture

```
Form (Nombre, Operación, Zona/Barrio, Presupuesto)
  ─▶ Airtable: create lead record
  ─▶ AI Agent (gpt-4.1-mini)
        ├─ memory: PostgreSQL chat memory (10 msgs, keyed by record id)
        ├─ tool: Supabase vector store (retrieve-as-tool, OpenAI embeddings)
        └─ output: Structured Output Parser {asunto_correo, cuerpo_correo}
  ─▶ Edit Fields (subject, body)
  ─▶ Gmail: send and WAIT for the reviewer's free-text answer
  ─▶ Text Classifier (gpt-4.1-mini): "Aprobado" | "Re-Generar"
        ├─ Aprobado   ─▶ Gmail: send the final email
        └─ Re-Generar ─▶ AI Agent 2: apply only the requested changes
                         ─▶ back to Edit Fields ─▶ review again
```

## Prerequisites

| Service | Purpose |
|---|---|
| n8n Form | Lead capture |
| Airtable | Lead record |
| OpenAI | Chat model (draft, classifier, rewrite) + embeddings |
| Supabase (pgvector) | Knowledge table queried as a retrieval tool |
| PostgreSQL | Chat memory |
| Gmail | Send-and-wait approval + final send |

## Key pieces

**Draft agent** — system prompt: write a persuasive email for the lead's operation (buy or rent), using the retrieved property context. Output is forced into JSON by the structured parser:
```json
{ "asunto_correo": "string", "cuerpo_correo": "string" }
```

**Human-in-the-loop** — Gmail *send and wait* emails the draft to the reviewer and pauses the execution until they reply in free text.

**Text Classifier** — reads the reply. *Aprobado* covers answers like "EXCELENTE", "VAMOS PARA ADELANTE", "ESTÁ OK", "MANDALO ASÍ"; anything with change requests → *Re-Generar*.

**Rewrite agent** — receives the current email plus the feedback; instructed to change **only** what the reviewer asked for. Same structured output, then the loop repeats until approved.

## Testing
Submit the form → approval email arrives → reply "cambiá el asunto y hacelo más corto" → a revised draft arrives → reply "ok mandalo" → final email is sent.

## Known limitations (prototype)
- Test data only: the property copy in the prompt is fictional and the final email goes to a test address.
- The Airtable mapping reuses columns from another table (e.g. budget stored in "Empresa").

## Stack
n8n · OpenAI (gpt-4.1-mini + embeddings) · Supabase vector store · PostgreSQL · Airtable · Gmail

## Status
Prototype with test data. Exported inactive.
