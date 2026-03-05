# Validation Sub-Workflow — HIPAA Compliance Amendments

**Companion to:** `validation-subworkflow-plan.md`
**Date:** March 4, 2026
**Purpose:** Compliance-driven amendments identified during HIPAA alignment review against 45 CFR Part 164. These notes supplement the sub-workflow design document without modifying its approved architecture.

---

## 1. Audit Log Schema Additions (Supplements Section 6) — IMPLEMENTED

Four new columns added to the AI Validation Audit Log Google Sheet:

| Column | Value | Populated By |
|---|---|---|
| Reviewed By | Name or email of the person who reviewed the flagged output | Review Response Handler (via email action buttons) |
| Review Date | ISO timestamp when review was submitted | Review Response Handler (auto-generated) |
| Disposition | Action taken: `Confirmed OK` / `Corrected` / `Escalated` / (blank if not yet reviewed) | Review Response Handler (via email action buttons) |
| Review Notes | Optional notes from the reviewer | Review Response Handler (via form submission) |

These columns are **not populated by the sub-workflow**. The sub-workflow appends only the automated columns (Timestamp through Execution ID). The review columns are populated via the **Review Response Handler** workflow (`workflows/Review Response Handler.json`), which serves an HTML form linked from action buttons in the review alert email.

**How it works:**
1. Review alert emails include three action buttons: **Confirmed OK** (green), **Corrected** (yellow), **Escalated** (red)
2. Clicking a button opens a web form (served by an n8n webhook) pre-filled with the execution ID and selected disposition
3. The reviewer enters their name and optional notes, then submits
4. The Review Response Handler updates the matching audit log row via Google Sheets Update (matched on Execution ID)

**Rationale:** Closes the audit loop required by §164.308(a)(1)(ii)(D) and §164.312(d) while keeping the process convenient — reviewers never need to open the Google Sheet directly.

---

## 2. ePHI Warning for Audit Log (Supplements Section 6) — CONFIRMED

The "Reasoning" column in the audit log will contain the validator's summary of its analysis. This summary may reference specific claims, names, dates, or medical information from the original input — making it **electronic protected health information (ePHI)**.

**BAA Coverage Confirmed:** The Google Sheets audit log (Sheet ID: `1XxE61qPZw0vDf50y4AnYbK-fOvpOxAXGdDRFkf5BYXo`) is stored in the BAA-covered Google Workspace account. The executed BAA with Google covers this sheet as an ePHI repository.

**Requirements:**
- The AI Validation Audit Log Google Sheet must be treated as an ePHI repository
- Access must be restricted to authorized personnel only (same standard as other PHI-containing systems)
- The sheet must be stored in the BAA-covered Google Workspace account
- Do not share the sheet link broadly or grant view-only access to staff who do not need it
- Include the sheet in regular access reviews

**Reference:** §164.312(a)(1) — Access Controls, §164.530(j) — Safeguards

---

## 3. Email Notification — Minimum Necessary Consideration (Supplements Section 7) — IMPLEMENTED

The AI output excerpt has been **removed from review alert emails entirely** and replaced with a link to the n8n execution log. Review emails now contain **zero PHI** — only metadata (confidence scores, flagged issues, severity levels) and an execution log link accessible only to authorized n8n users.

**Implementation:** The "Format Review Alert" node no longer truncates or includes `aiOutput`. Instead, it generates a "View Execution Log" button linking to:
```
https://simpledotbiz.app.n8n.cloud/workflow/{workflowId}/executions/{executionId}
```

**PHI risk status (all workflows):**

| Workflow | Previous PHI Risk | Current Status |
|---|---|---|
| Call Transcribe, Flag, Forward | HIGH | **MITIGATED** — no PHI in email |
| KOBE — Disability Content | MEDIUM | **MITIGATED** — no PHI in email |
| Veo3 Video Prompt | LOW | **MITIGATED** — no PHI in email |

**Reference:** §164.502(b) — Minimum Necessary Standard

---

## 4. Failure Mode Handling (Supplements Section 2) — IMPLEMENTED

**What happens when the validation sub-workflow fails:**

If this sub-workflow encounters an error (API timeout, Google Sheets write failure, n8n execution error):

1. **Parent workflow is unaffected** — the Execute Sub-Workflow call either returns an error payload or times out, and the parent's downstream logic continues with the original AI output
2. **No data is lost** from the parent workflow
3. **The failed execution is automatically backlogged** in the "Retry Queue" tab of the audit log spreadsheet
4. **A failure alert email** is sent to the `notifyEmail` address from the parent workflow

**Automated retry system:** The **Validation Retry Queue** workflow (`workflows/Validation Retry Queue.json`) handles error recovery:

| Component | Behavior |
|---|---|
| Error Trigger | Fires automatically when the sub-workflow fails; extracts original input parameters and logs to Retry Queue sheet |
| Failure Alert | Sends email to `notifyEmail` notifying that the execution failed and has been queued for retry |
| Scheduled Retry | Runs every 30 minutes; reads PENDING items from the Retry Queue sheet and re-executes the sub-workflow |
| Retry Limit | Maximum 3 attempts per execution |
| Escalation | After 3 failures, status is set to `FAILED_PERMANENT` and an escalation email is sent to `notifyEmail` |

**Retry Queue sheet columns:** Timestamp, Original Execution ID, Workflow Name, Status (`PENDING` / `RESOLVED` / `FAILED_PERMANENT`), Retry Count, Last Retry, Error Message, plus all original sub-workflow input parameters for replay.

**Setup requirement:** In the n8n UI, set the AI Validation Sub-Workflow's **Error Workflow** setting to point to the Validation Retry Queue workflow.

**Reference:** §164.308(a)(7) — Contingency Plan
