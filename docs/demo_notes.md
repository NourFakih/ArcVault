# Demo Notes

## Suggested Walkthrough

1. Show the Google Form or input sheet that feeds the workflow.
2. Show the n8n canvas and point out the trigger, OpenAI node, routing logic, Gmail escalation branch, and output nodes.
3. Submit or add a few sample rows.
4. Show `triage_records` being populated.
5. Show `escalations` being populated for flagged cases.
6. Show the Gmail escalation email for an incident or other escalated case.
7. Open the output CSV files included in `outputs/`.

## Screenshot Checklist

Save final screenshots with these names:

- `screenshots/workflow_canvas.png`
- `screenshots/google_form_input.png`
- `screenshots/triage_records_sheet.png`
- `screenshots/escalation_email.png`
- `screenshots/openai_route_nodes.png`

## Notes

- If you use Google Sheets directly as the visible input instead of the Google Form page, keep the filename `google_form_input.png` only if you are comfortable with that naming. Otherwise rename it and adjust your submission notes.
- If you take a final execution screenshot that clearly shows both OpenAI output and routing, `openai_route_nodes.png` is usually the most useful technical screenshot after the full workflow canvas.
