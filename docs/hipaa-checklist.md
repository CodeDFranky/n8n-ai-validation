# HIPAA Compliance Checklist — AI Workflow Deployment

**Reference for:** All n8n workflows processing client data through AI services.

---

## Before Deploying Any Workflow

- [ ] **BAA executed** with each AI provider used (OpenAI, Anthropic, Google, Deepgram)
- [ ] **BAA executed** with Google Workspace (covers Gmail and Google Sheets)
- [ ] **n8n instance access** restricted to authorized developers only
- [ ] **BAA scope covers validation sub-workflow** — GPT-4.1-mini receives `originalInput` (potentially full transcripts containing ePHI) for validation; confirm this data flow is explicitly within the OpenAI BAA scope — §164.314(a)
- [ ] **AI service inventory updated** — list the validation sub-workflow's AI model as a separate entry in the written inventory (it is a distinct data processor from the parent workflow's AI) — §164.308(a)(1)

## Data Minimization (Minimum Necessary Standard)

- [ ] Only pass the **minimum PHI needed** for the AI's task — avoid sending full transcripts or records when excerpts suffice
- [ ] Strip or redact PHI fields that are not relevant to the AI's function before passing data downstream
- [ ] **Validation sub-workflow inputs** — confirm that `originalInput` passed to the validator contains only the minimum PHI necessary for validation (not ancillary data) — §164.502(b)
- [ ] **Email notification truncation** — evaluate whether the 2,000-character AI output excerpt in review emails exposes PHI beyond what is necessary for the reviewer to act; adjust per workflow — §164.502(b)

## Encryption

- [ ] All AI API calls use **HTTPS/TLS** (n8n default — verify no custom HTTP overrides)
- [ ] Google Sheets audit log stored in **BAA-covered Google Workspace** (encrypted at rest by Google)
- [ ] Gmail notifications sent via **TLS** (verify recipient domains support TLS)

## Audit Log

- [ ] Audit log Google Sheet retained for **minimum 6 years** (check state requirements — some require 10+)
- [ ] Do not delete audit log rows — archive annually if needed
- [ ] Sheet access restricted to **authorized personnel only** (not shared broadly)
- [ ] **Audit log treated as ePHI** — the "Reasoning" column may contain PHI excerpts from the validator's analysis; apply the same access controls and retention rules as other ePHI repositories — §164.312(a)(1)
- [ ] **Audit log review cadence** — assign a specific person to review the validation audit log at least **monthly** (or quarterly for low-volume workflows); document the review date and reviewer name — §164.308(a)(1)(ii)(D)
- [ ] **Review closure tracking** — when a validation flags an output for human review, document who reviewed it, what action was taken, and the date; use a separate "Review Dispositions" tab or add columns to the audit log — §164.312(d)

## Validation Sub-Workflow Controls

- [ ] **Pre-production testing** — test sub-workflow with sample data before connecting to production workflows containing ePHI
- [ ] **Non-blocking risk acceptance** — document that the non-blocking design means downstream systems may act on AI output before validation completes; accept this as a documented risk — §164.308(a)(1)(ii)(A)
- [ ] **Downstream correction procedure** — for each parent workflow, define what happens if validation flags an output AFTER it has already been used downstream (e.g., email already sent, content already published); document the correction/retraction process — §164.308(a)(6)(ii)
- [ ] **Threshold change logging** — when adjusting a workflow's `strictnessThreshold`, record the old value, new value, date, reason, and who authorized the change

## Security Incident Procedures — §164.308(a)(6)

- [ ] **Incident escalation threshold** — if the validation audit log shows 3+ HIGH-severity failures for the same workflow within a 7-day period, treat as a potential security incident requiring investigation (not just individual email alerts)
- [ ] **Incident response contact** — designate who is responsible for investigating escalated validation failures (e.g., Franco R for technical, designated attorney for compliance)
- [ ] **Breach assessment procedure** — if a HIGH-severity validation failure involved incorrect PHI that was sent downstream, formally assess whether this constitutes a breach requiring notification under §164.402–408; document the assessment even if the conclusion is "no breach"

## Contingency Planning — §164.308(a)(7)

- [ ] **Validation sub-workflow failure mode** — if the sub-workflow errors (API timeout, Google Sheets unavailable, n8n crash), the parent workflow continues unaffected (by design); document this as accepted behavior and confirm parent workflows have error handling that does not depend on the sub-workflow returning successfully
- [ ] **Audit log backup** — include the AI Validation Audit Log Google Sheet in regular backup procedures (or confirm Google's built-in version history and Vault retention satisfy backup requirements)
- [ ] **Validator model unavailability** — if OpenAI's GPT-4.1-mini is unavailable for an extended period, define whether parent workflows continue without validation (current default) or whether manual review should be substituted

## Email Notifications

- [ ] Review alert recipients are **designated reviewers** only — not broad distribution lists
- [ ] Full AI output in emails limited to what's necessary for review
- [ ] Confirm recipient email accounts are on **BAA-covered domains**
- [ ] **Email content PHI spot-check** — periodically review alert emails to confirm the 2,000-character AI output excerpt does not include unnecessary PHI; for PHI-heavy workflows, consider reducing the character limit or switching to an n8n execution log link — §164.502(b)

## Ongoing

- [ ] Include all AI tools in your **annual HIPAA risk analysis**
- [ ] Maintain a **written inventory** of AI services that interact with ePHI
- [ ] Monitor and remediate vulnerabilities promptly
- [ ] Review and update this checklist when adding new workflows or AI providers
- [ ] **Validation audit log trend review** — during annual risk analysis, review validation trends (increasing HIGH-severity flags, consistently low-confidence workflows); use this data to adjust thresholds or modify prompts — §164.308(a)(1)(ii)(B)
- [ ] **Security awareness for review alerts** — ensure all staff who receive review alert emails understand what to do: review the output, document their disposition, escalate if needed — §164.308(a)(5)
