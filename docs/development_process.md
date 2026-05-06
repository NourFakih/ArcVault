# Development Process and AI Tool Usage

## Initial Planning

I started by reviewing the assessment requirements and discussing possible architectures with AI assistants. The main decision was to avoid over-engineering and keep the workflow close to the requested steps: ingestion, classification, enrichment, routing, structured output, and human escalation.

The assessment scenario is a simple but realistic agentic workflow: an inbound customer message arrives, an LLM classifies and enriches it, deterministic logic routes it, and high-risk or low-confidence cases are escalated for human review. Because of this, I avoided building a large multi-agent system and instead implemented a compact workflow with clear separation between language understanding and business rules.

## Tool Selection

I selected `n8n Cloud` as the orchestration layer because it provides visual workflow execution, trigger nodes, integration with Google Sheets, OpenAI API support, branching logic, and Gmail actions. I used Google Forms and Google Sheets for intake because this made it easy to simulate web-form submissions without building a custom frontend.

The final implementation uses:

- Google Form as the customer-facing intake surface.
- Google Sheets Trigger as the automatic ingestion mechanism.
- OpenAI structured output for classification, enrichment, and summary generation.
- n8n Code node for deterministic routing and escalation logic.
- Google Sheets as the persistent output destination.
- Gmail for human escalation notifications.

## AI-Assisted Development

I used AI tools during the build process, mainly for planning, prompt design, and workflow scaffolding. I first discussed the architecture with Claude and ChatGPT to keep the design aligned with the assessment and avoid overcomplicating the implementation. After that, I used Codex to scaffold parts of the repository and generate an initial n8n workflow export.

This was helpful for speed, but it also introduced some complexity that I had to manually correct. The initial scaffold mixed webhook-based and Google Sheets based approaches, which made the workflow more complicated than necessary. After rereading the assessment, I simplified the architecture and removed the webhook path, since Google Forms and Google Sheets already satisfied the automatic ingestion requirement.

## Manual Debugging and Refinement

Most of the implementation time was spent inside `n8n Cloud` connecting credentials, testing node outputs, and fixing mismatches between generated JSON, Google Sheets columns, and n8n item behavior. This was my first time using n8n, so a large part of the work was learning how n8n passes items between nodes, how Google Sheets Trigger behaves in test mode, and how to keep each row aligned with its corresponding OpenAI output.

The main issues I fixed manually were:

- Empty Google Form rows entering the workflow.
- Source and raw message fields becoming empty after OpenAI due to item-index mismatches.
- Identifier fields being returned inconsistently by the LLM.
- Message IDs being hard to manage inside n8n across batches and repeated test runs.
- Escalation rows needing to be sent both to a separate Google Sheets tab and by email.

To avoid wasting LLM tokens on invalid rows, I added a validation and filter step before the OpenAI call. To make the LLM output more reliable, I used strict structured output and a detailed prompt with fixed category labels, fixed priority labels, and a fixed identifiers schema.

## Message ID Design Choice

One issue that took longer than expected was sequential `message_id` generation. I initially tried to handle the ID in n8n using code, workflow state, or by reading the latest row from the output sheet. While this can work, it added unnecessary complexity for the assessment demo.

I eventually simplified the design by letting Google Sheets generate `message_id` using a formula, similar to an auto-increment field. This was the cleanest solution for the assessment because n8n only needs to append the structured triage row, while the sheet automatically assigns the visible sequential ID.

For production, I would not rely on a Google Sheets formula for ID generation. I would move this responsibility to a database, support-ticketing system, or backend service to guarantee atomic uniqueness under concurrent submissions.

## Cost and Prompt Tradeoff

I used a small OpenAI model to keep the workflow inexpensive. I considered making the prompt shorter, but the output needed to include classification, confidence, enrichment, identifiers, urgency, routing-supporting fields, and an internal summary in one call. A more detailed prompt was therefore useful to reduce output variability and avoid requiring multiple LLM calls.

The final design uses one OpenAI call per message instead of separate calls for classification, enrichment, and summarization. This reduces latency, cost, and workflow complexity, while still producing all required fields for the assessment.

## Final Workflow

The final workflow is:

`Google Form submission -> Google Sheets Trigger -> Normalize Input -> Filter Valid Rows -> OpenAI Triage -> Route + Escalate -> Append to triage_records -> IF escalation_flag = true -> Send Gmail escalation -> Append to escalations`

This keeps the workflow simple, explainable, and aligned with the assessment requirements. Codex was useful for scaffolding, but the generated workflow required manual review and simplification because some initial design choices introduced unnecessary complexity.
