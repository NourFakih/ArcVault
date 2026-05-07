# ArcVault AI Triage
ArcVault intake and triage assessment.

## Included

- `n8n/arcvault_intake_triage_workflow.json`
  The final n8n workflow export to submit. This package uses the version from `ArcVault Intake and Triage (1).json`.
- `docs/architecture_writeup.md`
  Short architecture and design rationale.
- `docs/prompts.md`
  The OpenAI prompt used for classification, enrichment, and summary generation, plus rationale.
- `docs/development_process.md`
  A short write-up of the implementation process, AI tool usage, and the main design decisions I kept or corrected manually.
- `outputs/triage_records.csv`
  Sample structured output records for the main queue.
- `outputs/escalations.csv`
  Sample structured output records for the escalation queue.
- `screenshots/`
  Place final screenshots here before zipping or sharing the submission.

## Workflow Summary

The workflow starts when a new row is added to the Google Sheets input source. It normalizes the inbound request, filters valid rows, sends the request to OpenAI for classification and enrichment, applies deterministic routing and escalation logic, appends the record to the triage sheet, and sends escalations to both the escalation sheet and Gmail.

High-level path:

`Google Sheets Trigger -> Normalize Input -> Filter Valid Rows -> OpenAI Triage -> Route + Escalate -> Append triage_records -> IF escalation_flag -> Send Gmail -> Append escalations`

## Development Process

I approached the assessment by first clarifying the required workflow and intentionally keeping the design small, since the assessment emphasized clarity over over-engineering. After comparing possible implementations, I chose an `n8n Cloud` workflow with Google Forms and Google Sheets as the intake layer, OpenAI for structured triage, deterministic routing logic in n8n, Google Sheets as the persistent output store, and Gmail for human escalation notifications.

The final workflow follows this path:

`Google Form submission -> Google Sheets Trigger -> Normalize Input -> Filter Valid Rows -> OpenAI Triage -> Route + Escalate -> Append to triage_records -> IF escalation_flag -> Send Gmail escalation + Append to escalations`

During development, I used AI tools to accelerate planning, prompt drafting, and workflow scaffolding. I manually reviewed and modified the generated workflow because some early scaffolding choices, especially mixing webhook and Google Sheets trigger patterns, created unnecessary complexity. The final implementation was simplified to a Google Forms and Google Sheets based workflow, which better matched the assessment requirement that the workflow starts automatically when a new message arrives.

One practical implementation choice was to keep message ID generation in Google Sheets using a formula rather than in n8n. This kept the n8n workflow focused on triage, routing, and escalation, while the sheet handled simple sequential IDs reliably for the demo. In production, I would move ID generation to a database, ticketing system, or backend service.

## Demo and Live Artifacts

### Screen Recording

A short screen recording is available here:

**YouTube Demo:** https://youtu.be/B6A5Z__VKRU 

Note: The recording has no audio and is intended as a visual walkthrough of the workflow. It is best viewed at **2x speed**.

The video shows:
- The n8n workflow execution.
- The Google Form intake submission.
- The linked Google Sheets response row.
- The OpenAI triage output.
- The `triage_records` output tab.
- The escalation path for human-review cases.
- The Gmail escalation notification.

### Live Google Form

The intake form used to submit sample ArcVault requests is available here:

**Google Form:** [\[ADD_GOOGLE_FORM_LINK_HERE\]](https://docs.google.com/forms/d/e/1FAIpQLSd0V06x33omApI_pKjYdFCzQJn5lx4k3vzIFX8i6Q3LNmqOEg/viewform)

### Live Google Sheet Output

The structured output records are available here:

**Google Sheet:** [\[ADD_GOOGLE_SHEET_LINK_HERE\]](https://docs.google.com/spreadsheets/d/1_cMyED6brUMe9dd1UBjv3TIXDnvsgQ0n-TbessHj4DY)

The Google Sheet contains:
- `incoming_requests`: form-submitted input messages.
- `triage_records`: structured classification, enrichment, routing, and escalation results.
- `escalations`: human-review cases only.

The `message_id` field is generated directly in Google Sheets using a formula, while n8n handles normalization, LLM triage, routing, escalation, and appending the final records.
