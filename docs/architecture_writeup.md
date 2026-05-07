# ArcVault AI Intake and Triage Workflow — Architecture Write-Up

## 1. System Design

The goal of this workflow is to simulate an AI-powered intake and triage system for ArcVault, an imaginary B2B software company receiving unstructured customer requests through channels such as email, web forms, and support portals. I implemented the solution as a compact n8n Cloud workflow because n8n provides a visual orchestration layer, trigger support, API integrations, conditional branching, and Google Sheets/Gmail actions without requiring a custom backend.

The workflow starts from a Google Form used as the customer-facing intake surface. Each form submission is automatically written to a linked Google Sheets response tab. n8n watches that sheet using a Google Sheets Trigger, so the workflow starts when a new row/message arrives. The incoming row is then normalized into a consistent structure containing the source, raw message, customer email, and form timestamp. A validation step filters out empty rows before the LLM call, which avoids wasting tokens on invalid inputs.

The normalized request is sent to an OpenAI model through a single structured-output call. This call performs classification, enrichment, identifier extraction, urgency analysis, and summary generation. I used one LLM call instead of multiple separate calls because the assessment is time-boxed, the input messages are short, and a single structured response reduces workflow complexity, latency, and cost.

After the OpenAI step, a Route + Escalate Code node applies deterministic business logic. This node maps the LLM classification to the correct destination queue and checks escalation criteria. The final structured record is appended to the `triage_records` Google Sheet tab. If the record is flagged for escalation, the workflow also sends a Gmail escalation email and appends the same case to an `escalations` tab.

State is held mainly in Google Sheets. The form response tab stores inbound requests, the `triage_records` tab stores final structured outputs, and the `escalations` tab stores human-review cases. The visible `message_id` is generated in Google Sheets using a formula, similar to an auto-increment field. I chose this for the assessment demo to keep n8n focused on triage, routing, and escalation instead of managing ID state inside workflow code.

Final workflow path:

Google Form submission  
→ Google Sheets Trigger  
→ Normalize Input  
→ Filter Valid Rows  
→ OpenAI Triage  
→ Route + Escalate  
→ Append to `triage_records`  
→ IF `escalation_flag = true`  
→ Send Gmail escalation  
→ Append to `escalations`

## 2. Routing Logic

The LLM classifies each request into one of five fixed categories: Bug Report, Feature Request, Billing Issue, Technical Question, or Incident/Outage. I intentionally constrained the model to fixed category labels because the next routing step needs predictable values.

The routing logic is implemented in the Route + Escalate node rather than left fully to the LLM. The LLM is used for language understanding, while n8n handles business rules. This makes the routing behavior easier to audit, debug, and change.

The queue mapping is:

- Bug Report → Engineering
- Feature Request → Product
- Billing Issue → Billing
- Technical Question → IT/Security
- Incident/Outage → Engineering by default, with escalation override to Human Review

This mapping keeps the system simple and reflects the likely ownership of each request type. Product owns feature requests, Billing owns invoice and contract-rate issues, IT/Security owns authentication and SSO questions, and Engineering owns product bugs and outages. A fallback route to Human Review is used when confidence is low or escalation criteria are met.

## 3. Escalation Logic

The escalation logic is intentionally simple. The goal was not to build a complex incident-management system, but to demonstrate that risky or ambiguous cases should not be handled automatically.

A record is escalated to Human Review when at least one of these conditions is true:

- The LLM confidence score is below 0.70.
- The category is Incident/Outage.
- The affected scope is `multiple_users`.
- The raw message contains outage-like language such as “outage,” “down for all users,” “multiple users affected,” “stopped loading,” or similar.
- The billing discrepancy is greater than 500.

When escalation is triggered, the destination queue is overridden to Human Review. The workflow also sends an email to the escalation reviewer with the classification, priority, confidence score, escalation reason, core issue, urgency signal, identifiers, original customer message, and internal summary.

I chose these criteria because they are easy to understand, deterministic, and directly tied to operational risk. Low-confidence cases should be reviewed because the model may be uncertain. Outage-like and multi-user cases should be reviewed quickly because they may represent active service disruption. Large billing discrepancies should be reviewed because they may create customer trust or financial issues.

## 4. Production-Scale Considerations

For the assessment, Google Forms, Google Sheets, n8n Cloud, OpenAI, and Gmail are enough to demonstrate the workflow clearly. In production, I would change several parts of the design.

First, I would replace Google Sheets with a real database, or support platform such as an internal case-management system. Google Sheets is useful for a demo, but it is not ideal for concurrent writes, audit controls, role-based access, or long-term operational reporting.

Second, I would move ID generation out of Google Sheets and into the backend system or database. The current formula-based message ID is simple and reliable for a demo, but production IDs should be generated atomically by the system of record to avoid race conditions.

Third, I would add stronger reliability controls: retries for failed API calls, dead-letter handling for failed messages, schema validation after the LLM response, workflow execution monitoring, and alerting when the workflow fails or produces invalid output.

Fourth, I would add privacy controls. For synthetic assessment data, using an external LLM API is acceptable. For real customer data, I would redact sensitive fields before sending requests to an external model, use approved enterprise data-retention settings, or route sensitive requests to a self-hosted/local model such as Qwen, or Llama if the company has the infrastructure. A private deployment could reduce third-party data exposure, but it would require more model evaluation, monitoring, and infrastructure management.

Fifth, I would evaluate cost and latency. The current design uses one LLM call per message, which is simple and has low orchestration overhead. If volume grows, I would benchmark smaller models, introduce caching for repeated questions, set API rate limits, and allow concurrent processing where safe. I would also monitor token usage and optimize the prompt, while keeping enough structure to maintain output quality.

Finally, I would consider splitting the LLM work only if needed. For this assessment, one structured-output call is enough. At larger scale, classification, entity extraction, summarization, and answer drafting could be separated into specialized model calls or smaller task-specific models. This could improve quality and allow cheaper models to handle simpler subtasks, but it would also increase orchestration complexity and latency.

## 5. Phase 2 Improvements

If I had another week, I would add several improvements.

First, I would add an evaluation set with expected labels and compare model outputs against expected category, priority, routing, escalation flag, and extracted identifiers. This would make prompt changes measurable instead of relying only on manual inspection.

Second, I would add a retrieval layer for frequent questions. For example, technical questions about Okta SSO could retrieve approved documentation before the LLM generates a summary or routing recommendation. This would improve consistency and reduce hallucination risk for repetitive support questions.

Third, I would add a human feedback loop. When a reviewer corrects a category, priority, or route, that feedback could be stored and later used to improve prompts, routing rules, or model fine-tuning.

Fourth, I would connect the workflow to a real ticketing or incident system. Escalated outage cases could create an incident ticket, billing cases could create a billing task, and feature requests could be logged into a product feedback board.

Fifth, I would add operational analytics: request volume by category, escalation rate, confidence distribution, average processing time, and common identifiers or product areas. This would help the business understand where requests are coming from and where automation is helping most.

Overall, the design prioritizes clarity, explainability, and safe automation. The LLM handles unstructured language understanding, while deterministic workflow logic handles routing and escalation. This separation keeps the system easier to debug and safer to operate.