# AI Output Validation Sub-Workflow — Design Document

**Date:** March 4, 2026
**Author:** Franco R & Team
**Status:** Proposal — Pending Implementation

---

## 1. Background & Problem

Hogan Smith Law runs multiple n8n automation workflows that use AI models (Claude Haiku, GPT-4.1-mini, Gemini 2.5-pro) to evaluate call transcripts, generate legal content, and create video production prompts. Currently, **none of these workflows validate AI outputs** for accuracy, hallucinations, or sourcing before the output is used downstream.

Given the legal domain we operate in, unverified AI outputs carry compliance risk — particularly around HIPAA, client confidentiality, and the accuracy of legal claims published on behalf of the firm.

### Original Directive

> "Franco R is tasked with building an AI confidence indicator and a recheck system to systematically validate AI outputs against hallucinations in areas of legal risk."

### Approach (Boss-Approved)

Lean into **Option 1: Source Enforcement** (double-pass AI content through a second AI that strictly cites where and how assumptions were made), combined with elements of **Option 2: Confidence Scoring + Human-in-the-Loop**.

- When a workflow is first released, validation is **strict** (flags most outputs for review).
- As the developer sees the AI consistently producing correct work, the threshold is **tuned looser** so it doesn't notify every time.
- The sub-workflow is **never a gate** — it audits and notifies but never blocks the parent workflow.

---

## 2. Design Principle

> **This sub-workflow is NOT a gate.** It never blocks, pauses, or prevents the parent workflow from continuing. The parent always proceeds with its original AI output regardless of the validation result. This is purely an **audit + notification layer**.

---

## 3. Sub-Workflow Architecture

A single reusable n8n sub-workflow, called from any parent workflow via an **Execute Sub-Workflow** node.

### 3.1 Node Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  1. Sub-Workflow Trigger                                            │
│     Receives input JSON from parent workflow                        │
│          │                                                          │
│          ▼                                                          │
│  2. Prepare Validation Context  [Code]                              │
│     Normalizes inputs into standard format                          │
│     Merges custom validation instructions if provided               │
│          │                                                          │
│          ▼                                                          │
│  3. Build Validation Prompt  [Code]                                 │
│     Constructs the source-enforcement prompt based on               │
│     task type: EVALUATION / GENERATION / CUSTOM                     │
│          │                                                          │
│          ▼                                                          │
│  4. LLM Model  [OpenAI GPT-4.1-mini, temperature=0]                │
│     ──connects to Node 5 via ai_languageModel──                     │
│          │                                                          │
│  5. Run Validation  [AI Agent]                                      │
│     Executes the validation prompt, retries on failure              │
│          │                                                          │
│          ▼                                                          │
│  6. Parse Validation Result  [Code]                                 │
│     Extracts structured JSON; handles parse failures gracefully     │
│          │                                                          │
│          ▼                                                          │
│  7. Apply Strictness Threshold  [Code]                              │
│     Compares confidence score against threshold                     │
│     Forces review if any HIGH-severity issues exist                 │
│          │                                                          │
│          ▼                                                          │
│  8. Write Audit Log  [Google Sheets Append]                         │
│     Logs EVERY result (pass or fail) to shared audit spreadsheet    │
│          │                                                          │
│          ▼                                                          │
│  9. Route Decision  [IF node]                                       │
│     needsHumanReview == true?                                       │
│          │                                                          │
│     ┌────┴────┐                                                     │
│     ▼         ▼                                                     │
│   TRUE      FALSE                                                   │
│     │         │                                                     │
│     ▼         │                                                     │
│  10. Format   │                                                     │
│   Review      │                                                     │
│   Alert       │                                                     │
│   [Code]      │                                                     │
│     │         │                                                     │
│     ▼         │                                                     │
│  11. Send     │                                                     │
│   Review      │                                                     │
│   Email       │                                                     │
│   [Gmail]     │                                                     │
│     │         │                                                     │
│     └────┬────┘                                                     │
│          ▼                                                          │
│  12. Return Results  [Code]                                         │
│     Returns structured JSON to parent workflow                      │
│     (both branches converge here)                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Input Parameters

These are passed by the parent workflow via the Execute Sub-Workflow node:

| Parameter | Required | Description |
|---|---|---|
| `workflowName` | YES | Human-readable name, e.g., "Call Transcribe Flag Forward" |
| `aiTaskType` | YES | `"EVALUATION"`, `"GENERATION"`, or `"CUSTOM"` |
| `originalInput` | YES | The raw data that was fed to the AI (transcript, question, case info) |
| `aiOutput` | YES | The AI's actual output being validated |
| `aiModel` | NO | Model name for the audit trail, e.g., "claude-haiku" |
| `aiPromptUsed` | NO | The prompt template used (helps the validator understand expectations) |
| `domainContext` | NO | Domain hint, e.g., "disability law", "call center evaluation" |
| `strictnessThreshold` | NO | Integer 0–100 (default: 80). Higher = stricter = more outputs flagged |
| `notifyEmail` | YES | Email address for human review alerts (mandatory — every workflow must specify a recipient) |
| `customValidationInstructions` | NO | Free-form validation criteria for CUSTOM tasks or to supplement built-in checks |

### 3.3 Return Payload

The parent workflow **always continues** after receiving this. It uses `originalAiOutput` for its normal downstream logic.

```json
{
  "passed": true,
  "needsHumanReview": false,
  "confidenceScore": 87,
  "overallAssessment": "PASS",
  "reviewLevel": "NONE",
  "decisionSummary": "PASSED: Score 87/75 threshold. No critical issues.",
  "reasoning": "The generated content accurately reflects...",
  "flaggedIssues": [],
  "sourceCitations": [
    {
      "claim": "SSDI benefits require 5 years of work credits",
      "source": "Input question: 'How many work credits for SSDI?'",
      "status": "PLAUSIBLE"
    }
  ],
  "unsourcedClaimCount": 0,
  "totalClaimsChecked": 12,
  "validationTimestamp": "2026-03-04T15:30:00.000Z",
  "strictnessThreshold": 75,
  "workflowName": "KOBE - Hogan Smith Automation",
  "originalAiOutput": "<the original AI output, passed through unchanged>"
}
```

---

## 4. Core Mechanism: Source Enforcement (Second AI Pass)

The second-pass prompt adapts based on `aiTaskType`:

### For EVALUATION Tasks (e.g., Call Transcribe)

The validator is told: *"For each criterion the original AI was asked to evaluate, cite the EXACT text from the original input that supports or contradicts the AI's conclusion."*

It checks for:
- **False positives** — AI flagged something that wasn't actually present
- **False negatives** — AI missed something it should have caught
- Whether evidence **strongly, moderately, or weakly** supports the conclusion

### For GENERATION Tasks (e.g., KOBE, Veo3)

The validator is told: *"For every factual claim in the generated content, cite WHERE that fact comes from in the original input or state 'NOT SOURCED — ASSUMED BY AI.'"*

Every claim is labeled:
- **VERIFIED** — traceable to the original input
- **PLAUSIBLE** — reasonable but not directly sourced
- **UNSOURCED** — no basis in the input (potential hallucination)

It specifically watches for hallucinated legal claims, statistics, names, dates, and procedures.

### For CUSTOM Tasks

When `aiTaskType` is `"CUSTOM"`, the `customValidationInstructions` string becomes the primary validation criteria. When used alongside EVALUATION or GENERATION, the custom instructions are appended as additional checks. This means **any future workflow can define its own validation rules without modifying the sub-workflow**.

### Structured Output

Both types produce the same JSON schema: source citations, flagged issues with severity (HIGH/MEDIUM/LOW), a confidence score (0–100), and an overall assessment (PASS/REVIEW_NEEDED/FAIL).

---

## 5. Strictness Threshold & Tuning

A single integer (0–100) controls how strict each workflow is. The logic:

- If `confidenceScore < strictnessThreshold` → flag for human review
- If **any HIGH-severity issue** exists → flag regardless of score (override)

### Recommended Tuning Schedule

| Phase | Threshold | What Happens |
|---|---|---|
| **Initial release** (week 1–2) | 85–95 | Nearly every AI output gets flagged. You're building a baseline. |
| **Building trust** (week 3–6) | 65–80 | Only medium/low-confidence outputs are flagged. |
| **Trusted operation** (month 2+) | 40–60 | Only genuinely suspicious outputs are flagged. |
| **Mature/stable** (month 3+) | 20–35 | Only clearly wrong outputs are flagged. |

To tune: change **one number** in the parent workflow's Execute Sub-Workflow node. No changes to the sub-workflow itself.

---

## 6. Audit Log

Every validation result — pass or fail — is appended to a shared Google Sheet. One row per validation.

| Column | Value |
|---|---|
| Timestamp | ISO datetime |
| Workflow Name | From input |
| Task Type | EVALUATION / GENERATION / CUSTOM |
| AI Model | From input |
| Confidence Score | 0–100 |
| Overall Assessment | PASS / REVIEW_NEEDED / FAIL |
| Strictness Threshold | The threshold used |
| Passed | TRUE / FALSE |
| Flagged Issues Count | Number of issues found |
| High Severity Count | Number of HIGH-severity issues |
| Unsourced Claims | Count |
| Total Claims Checked | Count |
| Reasoning | Validator's summary |
| Execution ID | n8n execution ID for traceability |

This provides a **permanent, queryable compliance trail**. Filter by workflow, sort by confidence score, track trends over time, and present during audits.

---

## 7. Human-in-the-Loop Notification

When `needsHumanReview` is true, a detailed email is sent containing:

- Confidence score and threshold
- Review severity level (CRITICAL / HIGH / MODERATE)
- All flagged issues with descriptions and recommendations
- Source citations (up to 10)
- The original AI output (first 2,000 characters)
- Execution ID for traceability

**Email subject format:** `[AI REVIEW HIGH] KOBE - Score 42/75`

The email is **non-blocking** — the parent workflow continues immediately. The notification exists so a human can retroactively review and correct if needed.

---

## 8. Integration with Existing Workflows

### 8.1 Workflow 1: Call Transcribe, Flag, Forward

**Current flow:**
```
Webhook → Deepgram Transcription → Claude Haiku (flag evaluation) → Filter → Gmail Alert
```

**Modified flow:**
```
Webhook → Deepgram Transcription → Claude Haiku → Execute Sub-Workflow → Filter → Gmail Alert
```

| Setting | Value |
|---|---|
| Insert point | Between Claude Haiku and Filter |
| Task type | `EVALUATION` |
| Starting threshold | `85` |
| Original input | The Deepgram transcript |
| AI output | Claude's TRUE/FALSE response |

**Behavior:** Flagged calls still get forwarded to attorneys immediately. The validation email goes to a separate channel as an audit trail.

---

### 8.2 Workflow 2: KOBE — Disability Content Generation

**Current flow:**
```
Webhook → Google Sheets → Filter → Loop → AI Agent (4 sections) → Parse → AI Agent1 (description) → Update Sheet
```

**Modified flow:**
```
... → AI Agent1 → Execute Sub-Workflow → Update Sheet (with extra "Validation Score" column)
```

| Setting | Value |
|---|---|
| Insert point | Between AI Agent1 and Append/Update Sheet, inside the loop |
| Task type | `GENERATION` |
| Starting threshold | `75` |
| Original input | The question from Google Sheets |
| AI output | Combined sections + description |

**Bonus:** Write `confidenceScore` to an extra Google Sheet column alongside the content for an inline audit trail.

---

### 8.3 Workflow 3: Veo3 Video Prompt Generation

**Current flow:**
```
Webhook → Gemini AI Agent → Respond to Webhook + Gmail
```

**Modified flow:**
```
Webhook → Gemini AI Agent → Execute Sub-Workflow → Respond to Webhook + Gmail
```

| Setting | Value |
|---|---|
| Insert point | Between Gemini AI Agent and Respond to Webhook |
| Task type | `GENERATION` |
| Starting threshold | `60` |
| Original input | Case info, character details, storyline from webhook body |
| AI output | The generated video prompts |

**Bonus:** Append validation score to the Gmail notification body.

---

## 9. Future-Proofing

| Concern | How It's Addressed |
|---|---|
| New workflow types | `customValidationInstructions` parameter lets any workflow define its own rules |
| New AI models | Validator model is configured in one node; swap anytime without touching the rest |
| New notification channels | Add Slack/Teams/webhook nodes alongside Gmail in the TRUE branch |
| Compliance audits | Audit log in Google Sheets provides permanent, queryable trail |
| Prompt versioning | Can be added later: log the validation prompt version in the audit sheet |
| Scaling | Each parent workflow call is independent; n8n handles concurrency |

---

## 10. Implementation Sequence

1. **Create the sub-workflow** as a new n8n workflow JSON file with all 12 nodes. Test with hardcoded sample inputs using n8n's pin data feature.
2. **Integrate with Veo3 first** — simplest flow, fewest nodes, lowest stakes.
3. **Integrate with Call Transcribe second** — different task type (EVALUATION) exercises the branching prompt logic.
4. **Integrate with KOBE last** — most complex flow with loop batching.
5. **Start all workflows at maximum strictness.** Monitor notification emails for 1–2 weeks. If the validation AI is consistently confirming the primary AI's outputs, lower each threshold by 10–15 points.

---

## 11. Resolved Decisions

- **Audit log location**: A new, dedicated Google Sheet ("AI Validation Audit Log") — separate from operational workflow sheets. All workflows log to the same sheet for a unified compliance view.
- **Notification email**: `notifyEmail` is **mandatory** (not optional). Every parent workflow must explicitly specify who receives review alerts.
- **Audit log content**: Metadata only in the sheet (scores, counts, reasoning summary). Full AI output is included in the notification email when review is flagged. For passed outputs, the n8n execution log serves as the detailed record.
- **Environment separation**: Not needed for now — one shared sub-workflow instance used across all workflows going forward.
