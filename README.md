# ArcVault AI Triage

This folder is the submission-ready package for the ArcVault intake and triage assessment.

## Included

- `n8n/arcvault_intake_triage_workflow.json`
  The final n8n workflow export to submit. This package uses the version from `ArcVault Intake and Triage (1).json`.
- `docs/architecture_writeup.md`
  Short architecture and design rationale.
- `docs/prompts.md`
  The OpenAI prompt used for classification, enrichment, and summary generation, plus rationale.
- `docs/development_process.md`
  A short write-up of the implementation process, AI tool usage, and the main design decisions I kept or corrected manually.
- `docs/demo_notes.md`
  Suggested demo flow and screenshot checklist.
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

## Before Submission

- Review `n8n/arcvault_intake_triage_workflow.json` and confirm it is the exact final export you want to share.
- Replace the sample screenshots folder contents with real screenshots from your final run.
- If needed, replace the sample CSV outputs with the latest exported outputs from your actual test run.

## Notes

- The files in `outputs/` are sample deliverables you can submit directly, but you can overwrite them with fresher exports if you run another final test.
- The `screenshots/` folder is intentionally left as a placeholder because screenshots need to come from your own final environment.
