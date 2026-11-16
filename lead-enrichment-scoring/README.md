# Lead Enrichment and AI Scoring (n8n, technical assessment)

Built for a technical assessment with **simulated leads** and placeholder APIs. Shows the batch pattern: fetch → loop → enrich → score with structured output → write to CRM → respond.

## Architecture

```
Webhook POST /enrich-leads (responds via node)
 ─▶ Fetch Leads (Code: 5 simulated CRM leads)
 ─▶ Loop (Split in Batches)
      ─▶ Enrich from API (GET /users/{id})
      ─▶ Score Lead (AI Agent, gpt-4o-mini, temp 0.2)
            └─ Structured Output Parser (auto-fix on)
      ─▶ Build Lead Record (Set)
      ─▶ Save to CRM (POST)
      ─▶ next lead
 ─▶ Respond to Webhook (all scored leads)
```

## Scoring
The agent scores each lead for a B2B SaaS company on company size (50–500 employees is the ideal profile), international presence and fit, and must return:
```json
{ "score": 85, "tier": "HOT", "rationale": "Mid-size tech company in Germany with international hiring needs" }
```
The structured parser has **auto-fix** enabled: malformed output is sent back to the model once to be corrected.

## Notes
- Enrichment and CRM endpoints are placeholders (`jsonplaceholder.typicode.com`); swap them for Clearbit/Apollo and your CRM API.
- Low temperature (0.2) keeps scores stable across runs.

## Stack
n8n · OpenAI (gpt-4o-mini) · REST APIs

## Status
Technical assessment with simulated data. Exported inactive.
