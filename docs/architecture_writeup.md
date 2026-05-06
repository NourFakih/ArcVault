# Architecture Write-Up

## System Design

The workflow is implemented in `n8n` and uses Google Sheets as the ingestion trigger and output store. A new inbound request arrives through the sheet that feeds the workflow, which allows the automation to start automatically without relying on a webhook endpoint. This is suitable for a lightweight take-home because it is easy to demonstrate and easy to inspect.

The flow is:

`Google Sheets Trigger -> Normalize Input -> Filter Valid Rows -> OpenAI Triage -> Route + Escalate -> Append triage_records -> IF escalation_flag -> Send Gmail -> Append escalations`

The input row is normalized first so the downstream prompt always receives a stable structure. OpenAI then performs the semantic work in a single call: category classification, priority assignment, confidence scoring, entity extraction, urgency assessment, and a short team-facing summary. After that, a deterministic code step applies routing rules and escalation logic so the final ownership decision is explicit and auditable instead of buried inside the model output.

## Routing Logic

The routing map is fixed and easy to explain:

- `Bug Report -> Engineering`
- `Feature Request -> Product`
- `Billing Issue -> Billing`
- `Technical Question -> IT/Security`
- `Incident/Outage -> Engineering`

This keeps ownership consistent and prevents the LLM from implicitly choosing teams on its own.

## Escalation Logic

A message is flagged for human review if any of the following are true:

- `confidence_score < 0.70`
- `category == "Incident/Outage"`
- `affected_scope == "multiple_users"`
- the raw message contains outage-style language
- `discrepancy_amount > 500`

When escalation is triggered, the record is still routed through the normal mapping so the standard team remains visible, but the workflow overrides the final destination to a human review path. In the current version, that means an escalation row is written to the `escalations` sheet and a Gmail notification is sent.

## Why This Design

I used one OpenAI step instead of splitting classification and enrichment into separate prompts because the assessment rewards clarity and intentionality more than raw component count. A single structured-output prompt reduces latency, reduces moving parts, and makes the workflow easier to reason about during a demo.

Google Sheets was chosen as both a trigger surface and a storage surface because it is simple, visual, and free to demonstrate. Gmail was used for escalation notification so a human-review action is visible in the workflow and not only stored as data.

## What I Would Change at Production Scale

- Move routing rules and escalation thresholds into configuration rather than hardcoding them.
- Add retries and failure handling around the OpenAI and Google nodes.
- Store records in a durable application database instead of Sheets.
- Add a feedback loop so corrected human-review outcomes improve prompt quality over time.
- Add request deduplication and better incident grouping for repeated outage reports.

## Phase 2

- Direct downstream integrations such as Jira or Helpdesk ticket creation.
- Better analytics over category mix, confidence distribution, and escalation rate.
- Multilingual handling for non-English inbound requests.
- Support for richer attachments or screenshots in the ingestion stage.
