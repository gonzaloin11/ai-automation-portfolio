# Slack Thread Summarizer — Batch Classification (n8n, technical assessment)

Built for a technical assessment with **mock data**: 100+ Slack threads across dozens of channels arrive in one webhook call; the workflow cleans them, classifies and summarizes them **in batches of 10** with a strict JSON schema, and posts each result.

## Architecture

```
Webhook POST /slack-thread
 ─▶ Clean Messages (Code): unwrap body[], strip <@mentions>, <#channels>, link markup
 ─▶ Loop Over Items (batch size 10)
      ─▶ AI Agent "Batch Summarizer" (gpt-4.1-mini)
            └─ Structured Output Parser (tickets[] schema)
      ─▶ Code: safe-parse output, split tickets into items
      ─▶ Check Data Received ── ok ─▶ Format Output ─▶ next batch
                              └─ empty ─▶ Send error to Slack (#alerts-errors)
 ─▶ (done) Send to Slack
```

## Output schema (per ticket)

| Field | Type / values |
|---|---|
| `ticketId` | `TICKET-{n}` |
| `index`, `channel` | position and channel |
| `category` | PAYROLL · BENEFITS · COMPLIANCE · ONBOARDING · TAX · HR · IT · FINANCE · LEGAL · OTHER |
| `urgency` | LOW · MEDIUM · HIGH · CRITICAL |
| `summary` | short executive summary |
| `actionItems`, `risks` | arrays |
| `keyEntities` | object (people, dates, amounts…) |
| `originalBody` | the cleaned thread |

## Design decisions
- **Batching (10 threads per call)** cuts the number of LLM calls by about 10× versus one call per thread.
- **Deterministic cleanup before the model:** Slack markup is removed in code, not by the LLM.
- **Defensive parsing:** the Code node strips code fences and accepts several shapes (`tickets`, arrays, nested `output`) before splitting.
- **Error path:** if a batch returns no summary, an alert is posted with the error and timestamp instead of failing silently.

Outbound "Slack" calls point to a placeholder API (`jsonplaceholder.typicode.com`) because the assessment used mock endpoints.

## Stack
n8n · OpenAI (gpt-4.1-mini) · Slack (HTTP)

## Status
Technical assessment with mock data. Exported inactive.
