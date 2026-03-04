# Validation Sub-Workflow — HIPAA Compliance Amendments

**Companion to:** `validation-subworkflow-plan.md`
**Date:** March 4, 2026
**Purpose:** Compliance-driven amendments identified during HIPAA alignment review against 45 CFR Part 164. These notes supplement the sub-workflow design document without modifying its approved architecture.

---

## 1. Audit Log Schema Additions (Supplements Section 6)

Add three manual-entry columns to the AI Validation Audit Log Google Sheet:

| Column | Value | Populated By |
|---|---|---|
| Reviewed By | Name or email of the person who reviewed the flagged output | Human reviewer (manual) |
| Review Date | Date the review was completed | Human reviewer (manual) |
| Disposition | Action taken: `Confirmed OK` / `Corrected` / `Escalated` / (blank if not yet reviewed) | Human reviewer (manual) |

These columns are **not populated by the sub-workflow**. The sub-workflow appends only the automated columns (Timestamp through Execution ID). The review columns are blank on initial write and filled by the human reviewer after receiving a review alert email.

**Rationale:** Closes the audit loop required by §164.308(a)(1)(ii)(D) and §164.312(d). Without these columns, there is no record that flagged items were actually reviewed or what action was taken.

---

## 2. ePHI Warning for Audit Log (Supplements Section 6)

The "Reasoning" column in the audit log will contain the validator's summary of its analysis. This summary may reference specific claims, names, dates, or medical information from the original input — making it **electronic protected health information (ePHI)**.

**Requirements:**
- The AI Validation Audit Log Google Sheet must be treated as an ePHI repository
- Access must be restricted to authorized personnel only (same standard as other PHI-containing systems)
- The sheet must be stored in the BAA-covered Google Workspace account
- Do not share the sheet link broadly or grant view-only access to staff who do not need it
- Include the sheet in regular access reviews

**Reference:** §164.312(a)(1) — Access Controls, §164.530(j) — Safeguards

---

## 3. Email Notification — Minimum Necessary Consideration (Supplements Section 7)

The sub-workflow design includes the first 2,000 characters of the original AI output in review alert emails. This is a **fixed truncation** and does not filter or redact PHI.

**Per-workflow evaluation recommended:**

| Workflow | PHI Risk in AI Output | Recommendation |
|---|---|---|
| Call Transcribe, Flag, Forward | **HIGH** — AI output may reference caller names, medical conditions, case details | Consider reducing excerpt to 500 chars or replacing with n8n execution log link |
| KOBE — Disability Content | **MEDIUM** — Generated content may reference disability types, legal claims | 2,000 chars likely acceptable; spot-check periodically |
| Veo3 Video Prompt | **LOW** — Video scene descriptions typically do not contain PHI | 2,000 chars acceptable |

If a workflow's AI output routinely contains PHI, the `notifyEmail` alert should include only enough context for the reviewer to locate and assess the output — not the full output itself. A link to the n8n execution log (accessible only to authorized users) satisfies the review need without transmitting PHI via email.

**Reference:** §164.502(b) — Minimum Necessary Standard

---

## 4. Failure Mode Documentation (Supplements Section 2)

**What happens when the validation sub-workflow fails:**

If this sub-workflow encounters an error (API timeout, Google Sheets write failure, n8n execution error):

1. **Parent workflow is unaffected** — the Execute Sub-Workflow call either returns an error payload or times out, and the parent's downstream logic continues with the original AI output
2. **No data is lost** from the parent workflow
3. **The audit record for that execution will be missing** from the validation audit log

**Monitoring recommendation:** Periodically compare the validation audit log row count against parent workflow execution counts (available in n8n's execution log). A significant discrepancy indicates the sub-workflow has been failing silently. This check should be part of the monthly audit log review.

**Accepted risk:** During a sub-workflow outage, parent workflows continue to operate without validation. This is by design (non-blocking architecture). If extended outages are observed, consider whether manual review should substitute until the sub-workflow is restored.

**Reference:** §164.308(a)(7) — Contingency Plan
