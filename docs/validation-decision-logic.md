# AI Validation Decision Logic

**Date:** March 5, 2026
**Author:** Franco R & Team

---

## 1. Pipeline Overview

When any workflow uses AI to generate or evaluate content, the AI Validation Sub-Workflow acts as a **second-pass auditor**. It never blocks the parent workflow — it audits and notifies, but the parent always continues with its original AI output.

```
Parent Workflow produces AI output
        |
        v
AI Validation Sub-Workflow (second AI checks the first AI's work)
        |
        v
Apply Strictness Threshold (pass/fail/review decision)
        |
        v
Log to Audit Sheet (every result, pass or fail)
        |
        v
Route Decision: needs human review?
       / \
     YES   NO
      |     |
      v     |
  Send email |
  with       |
  "Review    |
  Now"       |
  button     |
      |     |
      v     v
  Return Results to parent workflow
  (parent continues either way)
```

---

## 2. What the Validator Checks (by Task Type)

The second-pass AI (GPT-4.1-mini, temperature=0) adapts its validation based on the type of AI task being audited:

### GENERATION Tasks (e.g., KOBE disability content)

For every factual claim in the AI output, the validator checks: **can it be traced back to the original input?**

Each claim is labeled:
- **VERIFIED** — directly traceable to the original input
- **PLAUSIBLE** — reasonable inference but not directly sourced
- **UNSOURCED** — no basis in the input (potential hallucination)

Special attention to: hallucinated legal claims, statistics, attorney names, dates, case numbers, and office locations.

### EVALUATION Tasks (e.g., Call Transcribe flag evaluation)

The validator checks whether the AI's TRUE/FALSE evaluation was correct by looking for:
- **False positives** — AI flagged something that wasn't actually present in the transcript
- **False negatives** — AI missed something it should have caught
- Evidence strength: does the transcript **strongly, moderately, or weakly** support the AI's conclusion?

### CUSTOM Tasks (e.g., Veo3 video prompt validation)

Uses the `customValidationInstructions` provided by the parent workflow as the primary validation criteria. For example, Veo3 prompts are checked against rules like:
- Never use the words "us", "we", or "defense"
- Always refer to the firm in third person
- Include the instruction to never show text overlays, subtitles, or logos
- Script is approximately 20 words for an 8-second clip

---

## 3. Validator Output Fields

The second-pass AI returns a structured JSON with these fields:

| Field | Type | Description |
|---|---|---|
| `confidenceScore` | Integer 0-100 | How confident the validator is that the AI output is correct |
| `overallAssessment` | String | `PASS`, `REVIEW_NEEDED`, or `FAIL` |
| `flaggedIssues` | Array | Each issue has: `issue` (description), `severity` (HIGH/MEDIUM/LOW), `recommendation` |
| `sourceCitations` | Array | Each citation has: `claim` (what the AI said), `source` (where it came from or NOT SOURCED), `status` (VERIFIED/PLAUSIBLE/UNSOURCED) |
| `reasoning` | String | Summary of validation findings |

---

## 4. The Decision Logic (Apply Strictness Threshold)

This is the core decision point. Two things are evaluated:

### Primary Check: Score vs. Threshold

```
passed = (confidenceScore >= strictnessThreshold)
needsHumanReview = NOT passed
```

If the validator's confidence score meets or exceeds the threshold, the output passes. If it falls below, it gets flagged for human review.

### Override: HIGH-Severity Issues

```
If ANY flagged issue has severity = "HIGH" → needsHumanReview = true
```

**This override is absolute.** Even if the confidence score is 95 out of a threshold of 80, a single HIGH-severity issue forces human review. This ensures things like hallucinated attorney names or false legal claims always get a human look.

### Combined Logic

| Confidence Score | HIGH-Severity Issues | Result |
|---|---|---|
| >= threshold | 0 | **PASSED** — no review needed |
| >= threshold | 1+ | **FLAGGED** — HIGH issue override |
| < threshold | 0 | **FLAGGED** — score too low |
| < threshold | 1+ | **FLAGGED** — both triggers |

---

## 5. Review Level Assignment

When `needsHumanReview` is true, a review level is assigned to indicate urgency:

| Condition | Review Level | Meaning |
|---|---|---|
| Score < 50% of threshold, OR 2+ HIGH issues | **CRITICAL** | Serious problems, likely needs correction |
| 1 HIGH issue, OR score < 75% of threshold | **HIGH** | Significant concern, review promptly |
| Everything else that's flagged | **MODERATE** | Minor concerns, review when available |
| Not flagged at all | **NONE** | Output passed validation |

### Examples (with threshold = 80):

- Score 30 → below 40 (50% of 80) → **CRITICAL**
- Score 55 with 1 HIGH issue → **HIGH**
- Score 72 with 0 HIGH issues → below 60 (75% of 80) → **HIGH**
- Score 75 with 0 HIGH issues → **MODERATE**
- Score 85 with 0 HIGH issues → **NONE** (passed)
- Score 90 with 1 HIGH issue → override kicks in → **HIGH**

---

## 6. The Three Outcome Paths

### Path A: PASSED (needsHumanReview = false)

- Confidence score >= threshold AND zero HIGH-severity issues
- Result logged to Google Sheets audit log
- No email sent
- Parent workflow continues normally
- The `decisionSummary` reads: "PASSED: Score X/Y threshold. No critical issues."

### Path B: FLAGGED for Human Review (needsHumanReview = true)

- Confidence score < threshold OR any HIGH-severity issue exists
- Result logged to Google Sheets audit log
- Email sent to `notifyEmail` address containing:
  - Confidence score and threshold
  - Review severity level (CRITICAL / HIGH / MODERATE)
  - All flagged issues with descriptions and recommendations
  - Source citations (up to 10)
  - Link to n8n execution log
  - **"Review Now" button** → opens a form where reviewer selects disposition, enters name, and adds notes
- Reviewer's response updates the audit log row with: Reviewed By, Review Date, Disposition, Review Notes
- **Parent workflow still continues** — the email is non-blocking
- The `decisionSummary` reads: "FLAGGED: Score X/Y threshold. N high-severity issue(s). Review level: Z."

### Path C: ERROR (parse failure)

- If the validator AI returns unparseable JSON
- Defaults to: `confidenceScore: 50`, `overallAssessment: REVIEW_NEEDED`
- Creates a flagged issue: "Validation response could not be parsed"
- This typically triggers human review since 50 falls below most thresholds
- Ensures failures are never silently ignored

---

## 7. Strictness Threshold Tuning

The `strictnessThreshold` parameter (0-100) is set per workflow in the parent's Execute Sub-Workflow node. One number to change, no modifications to the sub-workflow itself.

| Phase | Recommended Threshold | What Happens |
|---|---|---|
| **Initial release** (week 1-2) | 85-95 | Nearly every AI output gets flagged. Building a baseline. |
| **Building trust** (week 3-6) | 65-80 | Only medium/low-confidence outputs are flagged. |
| **Trusted operation** (month 2+) | 40-60 | Only genuinely suspicious outputs are flagged. |
| **Mature/stable** (month 3+) | 20-35 | Only clearly wrong outputs are flagged. |

---

## 8. Test Harness Scenarios

The test harness runs 6 scenarios to verify the decision logic works correctly:

| # | Scenario | Type | Expected | Why |
|---|---|---|---|---|
| 1 | Clean KOBE content, all facts traceable to input | GENERATION | PASS | Score should be high, no HIGH issues |
| 2 | Hallucinated stats ($12K average), fake attorney (John Williams), fake law (Section 14-7) | GENERATION | FLAG | Unsourced claims trigger HIGH issues |
| 3 | Correctly flagged upset caller + Dan Smith mention as TRUE | EVALUATION | PASS | Validator confirms TRUE was the right call |
| 4 | Calm routine call (Sarah Johnson status check) wrongly flagged as TRUE | EVALUATION | FLAG | False positive detected — no upset caller, no Dan Smith |
| 5 | Veo3 prompt follows all custom rules (third person, no banned words, short script) | CUSTOM | PASS | No rule violations found |
| 6 | Veo3 prompt uses "we" and "defense", has logo on shirt, script is 40+ words | CUSTOM | FLAG | Multiple custom rule violations |

---

## 9. Audit Log Columns

Every validation result (pass or fail) is appended to the Google Sheet audit log:

| Column | Source |
|---|---|
| Timestamp | Generated at validation time |
| Workflow Name | From parent workflow input |
| Task Type | EVALUATION / GENERATION / CUSTOM |
| AI Model | From parent workflow input |
| Confidence Score | 0-100 from validator |
| Overall Assessment | PASS / REVIEW_NEEDED / FAIL |
| Strictness Threshold | The threshold used for this run |
| Passed | TRUE / FALSE |
| Flagged Issues Count | Number of issues the validator found |
| High Severity Count | Number of HIGH-severity issues |
| Unsourced Claims | Count of UNSOURCED citations |
| Total Claims Checked | Total citations evaluated |
| Reasoning | Validator's summary explanation |
| Execution ID | n8n execution ID for traceability |

When a human reviewer submits a response via the review form, these additional columns are updated on the same row:

| Column | Source |
|---|---|
| Reviewed By | Name or email from the review form |
| Review Date | Timestamp of form submission |
| Disposition | Confirmed OK / Corrected / Escalated |
| Review Notes | Free-text notes from reviewer |

---

## Key Takeaway

This system is an **audit and notification layer, not a gate**. The parent workflow always proceeds with its original AI output. The validation exists so that:

1. Every AI output has a permanent compliance trail in the audit log
2. Suspicious outputs get flagged for human review via email
3. Humans can retroactively review and correct if needed
4. The strictness can be tuned per workflow as trust builds over time
