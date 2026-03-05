# AI Validation Sub-Workflow — Reference Guide

**File:** `workflows/AI Validation Sub-Workflow.json`
**Status:** Implemented
**n8n Instance:** `simpledotbiz.app.n8n.cloud`

---

## Overview

A reusable 12-node n8n sub-workflow that audits AI-generated outputs for hallucinations, unsourced claims, and legal compliance risks. It is called from parent workflows via an **Execute Sub-Workflow** node.

**This is NOT a gate.** It never blocks, pauses, or prevents the parent workflow from continuing. It is purely an audit + notification layer.

### How It Works

1. Parent workflow sends AI output + context to the sub-workflow
2. A second AI pass (GPT-4.1-mini at temperature=0) validates every claim against the source material
3. Results are scored, logged to Google Sheets, and optionally emailed for human review
4. The original AI output is returned unchanged to the parent workflow

---

## Node Pipeline

```
Trigger → Prepare Context → Build Prompt → AI Agent ← OpenAI Model (gpt-4.1-mini, temp=0)
                                              ↓
                                    Parse Result → Apply Threshold → Write Audit Log → Route Decision
                                                                                          ↓
                                                                    TRUE:  Format Alert → Send Email → Return Results
                                                                    FALSE: ──────────────────────────→ Return Results
```

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | Sub-Workflow Trigger | executeWorkflowTrigger | Receives input JSON from parent |
| 2 | Prepare Validation Context | Code | Normalizes inputs, sets defaults, validates required fields |
| 3 | Build Validation Prompt | Code | Constructs task-specific source-enforcement prompt |
| 4 | OpenAI Chat Model | lmChatOpenAi | GPT-4.1-mini at temperature=0 |
| 5 | Run Validation | AI Agent | Executes the validation prompt (retries on failure) |
| 6 | Parse Validation Result | Code | Extracts structured JSON from AI response |
| 7 | Apply Strictness Threshold | Code | Compares score vs threshold, forces review on HIGH issues |
| 8 | Write Audit Log | Google Sheets (append) | Logs every result to the audit spreadsheet |
| 9 | Route Decision | IF | Routes based on `needsHumanReview` |
| 10 | Format Review Alert | Code | Builds HTML email with issues, citations, AI output |
| 11 | Send Review Email | Gmail | Sends alert to `notifyEmail` address |
| 12 | Return Results | Code | Returns structured payload to parent workflow |

---

## Input Parameters

Pass these via the Execute Sub-Workflow node in the parent workflow:

| Parameter | Required | Type | Default | Description |
|---|---|---|---|---|
| `workflowName` | Yes | string | — | Human-readable name, e.g., `"KOBE - Hogan Smith Automation"` |
| `aiTaskType` | Yes | string | — | `"EVALUATION"`, `"GENERATION"`, or `"CUSTOM"` |
| `originalInput` | Yes | string | — | The raw data fed to the AI (transcript, question, case info) |
| `aiOutput` | Yes | string | — | The AI's actual output being validated |
| `notifyEmail` | Yes | string | — | Email for human review alerts |
| `aiModel` | No | string | `"unknown"` | Model name for audit trail |
| `aiPromptUsed` | No | string | `""` | The prompt template used |
| `domainContext` | No | string | `"general"` | Domain hint, e.g., `"disability law"` |
| `strictnessThreshold` | No | integer | `80` | 0–100. Higher = stricter = more outputs flagged |
| `customValidationInstructions` | No | string | `""` | Free-form validation criteria for CUSTOM tasks |

---

## Task Types

### EVALUATION
For workflows where the AI evaluates/classifies input (e.g., Call Transcribe flagging calls as TRUE/FALSE).

Checks for:
- **False positives** — AI flagged something not actually present
- **False negatives** — AI missed something it should have caught
- Evidence strength: STRONGLY / MODERATELY / WEAKLY supports the conclusion

### GENERATION
For workflows where the AI generates content (e.g., KOBE SEO content, Veo3 video prompts).

Every factual claim is labeled:
- **VERIFIED** — traceable to the original input
- **PLAUSIBLE** — reasonable but not directly sourced
- **UNSOURCED** — no basis in the input (potential hallucination)

Watches specifically for hallucinated legal claims, statistics, names, dates, and procedures.

### CUSTOM
Uses `customValidationInstructions` as the primary validation criteria. For future workflows with unique validation needs.

---

## Return Payload

The parent workflow receives this JSON. It should use `originalAiOutput` for its normal downstream logic.

```json
{
  "passed": true,
  "needsHumanReview": false,
  "confidenceScore": 87,
  "overallAssessment": "PASS",
  "reviewLevel": "NONE",
  "decisionSummary": "PASSED: Score 87/80 threshold. No critical issues.",
  "reasoning": "The generated content accurately reflects...",
  "flaggedIssues": [],
  "sourceCitations": [
    {
      "claim": "SSDI benefits require work credits",
      "source": "Input question: 'How many work credits for SSDI?'",
      "status": "PLAUSIBLE"
    }
  ],
  "unsourcedClaimCount": 0,
  "totalClaimsChecked": 12,
  "validationTimestamp": "2026-03-04T15:30:00.000Z",
  "strictnessThreshold": 80,
  "workflowName": "KOBE - Hogan Smith Automation",
  "originalAiOutput": "<the original AI output, passed through unchanged>"
}
```

---

## Strictness Threshold Logic

- `confidenceScore >= strictnessThreshold` → **PASS**
- `confidenceScore < strictnessThreshold` → **FLAGGED** for human review
- Any **HIGH-severity** issue → **forces human review** regardless of score

### Review Levels

| Level | Condition |
|---|---|
| CRITICAL | Score < 50% of threshold OR 2+ HIGH-severity issues |
| HIGH | 1+ HIGH-severity issue OR score < 75% of threshold |
| MODERATE | Score below threshold but no HIGH-severity issues |
| NONE | Passed with no issues |

### Recommended Tuning Schedule

| Phase | Threshold | Purpose |
|---|---|---|
| Initial release (week 1–2) | 85–95 | Build a baseline; nearly everything is flagged |
| Building trust (week 3–6) | 65–80 | Only medium/low-confidence outputs flagged |
| Trusted operation (month 2+) | 40–60 | Only genuinely suspicious outputs flagged |
| Mature/stable (month 3+) | 20–35 | Only clearly wrong outputs flagged |

---

## Audit Log

Every validation (pass or fail) appends a row to the **AI Validation Audit Log** Google Sheet.

| Column | Source |
|---|---|
| Timestamp | ISO datetime of validation |
| Workflow Name | `workflowName` input |
| Task Type | EVALUATION / GENERATION / CUSTOM |
| AI Model | `aiModel` input |
| Confidence Score | 0–100 from validator |
| Overall Assessment | PASS / REVIEW_NEEDED / FAIL |
| Strictness Threshold | Threshold used |
| Passed | TRUE / FALSE |
| Flagged Issues Count | Number of issues found |
| High Severity Count | Number of HIGH-severity issues |
| Unsourced Claims | Count of UNSOURCED citations |
| Total Claims Checked | Count of all citations |
| Reasoning | Validator's summary |
| Execution ID | n8n execution ID |
| Reviewed By | Name/email of reviewer (populated via Review Response Handler) |
| Review Date | ISO timestamp of review submission |
| Disposition | `Confirmed OK` / `Corrected` / `Escalated` |
| Review Notes | Optional reviewer notes |

---

## Email Alerts

When `needsHumanReview` is true, an HTML email is sent containing:

- **Quick Review Action Buttons** — Confirmed OK (green), Corrected (yellow), Escalated (red)
- Confidence score vs threshold
- Review severity level (CRITICAL / HIGH / MODERATE)
- All flagged issues with severity and recommendations
- Source citations (up to 10)
- **Link to n8n execution log** (no PHI in email — accessible only to authorized n8n users)
- Execution ID for traceability

Clicking a review button opens a web form (served by the Review Response Handler workflow) where the reviewer enters their name and optional notes. The form submission updates the audit log row directly.

**Subject format:** `[AI REVIEW HIGH] KOBE - Hogan Smith Automation - Score 42/75`

**Sent via:** Kayser Gmail credential
**Sent to:** The `notifyEmail` address provided by the parent workflow

---

## Credentials

| Node | Credential | ID |
|---|---|---|
| OpenAI Chat Model | OpenAi account (`openAiApi`) | `NpwqDbmTFAPZiqsb` |
| Write Audit Log | Google Sheets Account 2 (`googleSheetsOAuth2Api`) | `O4aWPntpGBnCjIQl` |
| Send Review Email | thomas@simple.biz (`gmailOAuth2`) | `szVBtjN1lnbrsjaj` |

---

## Setup Instructions

### 1. Create the Audit Log Sheet

Create a new Google Sheet titled **"AI Validation Audit Log"** with these column headers in row 1:

```
Timestamp | Workflow Name | Task Type | AI Model | Confidence Score | Overall Assessment | Strictness Threshold | Passed | Flagged Issues Count | High Severity Count | Unsourced Claims | Total Claims Checked | Reasoning | Execution ID
```

### 2. Update the Sheet ID

Copy the Sheet ID from the URL (the long string between `/d/` and `/edit`) and replace `REPLACE_WITH_AUDIT_LOG_SHEET_ID` in the workflow JSON, or update it in the n8n UI after import.

### 3. Import into n8n

Import `AI Validation Sub-Workflow.json` into the n8n Cloud instance at `simpledotbiz.app.n8n.cloud`.

### 4. Verify Credentials

In the n8n editor, open each credentialed node and confirm the credentials are connected:
- OpenAI Chat Model → OpenAi account
- Write Audit Log → Google Sheets Account 2
- Send Review Email → thomas@simple.biz

### 5. Test

Use n8n's "Test workflow" with the included pin data (a sample GENERATION task with a hallucinated statistic). Verify:
- A row appears in the audit log sheet
- An email is sent (the sample should trigger review)
- The return payload includes `originalAiOutput` unchanged

---

## Integrating with Parent Workflows

Add an **Execute Sub-Workflow** node in the parent workflow. The sub-workflow is non-blocking — the parent always continues with its original AI output.

### Call Transcribe (EVALUATION)

Insert between Claude Haiku and Filter:

```json
{
  "workflowName": "Call Transcribe Flag Forward",
  "aiTaskType": "EVALUATION",
  "originalInput": "{{ $('HTTP Request').item.json.results.channels[0].alternatives[0].transcript }}",
  "aiOutput": "{{ $json.content[0].text }}",
  "aiModel": "claude-haiku-4-5",
  "domainContext": "call center evaluation",
  "strictnessThreshold": 85,
  "notifyEmail": "franco@hogansmith.com"
}
```

### KOBE Content Generation (GENERATION)

Insert between AI Agent1 and Append/Update Sheet, inside the loop:

```json
{
  "workflowName": "KOBE - Hogan Smith Automation",
  "aiTaskType": "GENERATION",
  "originalInput": "{{ $('Filter').item.json.Question }}",
  "aiOutput": "{{ $json.output }}",
  "aiModel": "gpt-4.1-mini",
  "domainContext": "disability law",
  "strictnessThreshold": 75,
  "notifyEmail": "franco@hogansmith.com"
}
```

### Veo3 Video Prompts (GENERATION)

Insert between Gemini AI Agent and Respond to Webhook:

```json
{
  "workflowName": "Veo3 Hogan Smith Social Media Prompt",
  "aiTaskType": "GENERATION",
  "originalInput": "{{ JSON.stringify($('Webhook').item.json.body) }}",
  "aiOutput": "{{ $json.output }}",
  "aiModel": "gemini-2.5-pro",
  "domainContext": "video production for disability law firm",
  "strictnessThreshold": 60,
  "notifyEmail": "franco@hogansmith.com"
}
```

---

## Error Handling

- **Missing required fields:** The Prepare Validation Context node returns an error payload with `passed: true` and `overallAssessment: "ERROR"` so the parent workflow is never blocked.
- **AI response parse failure:** The Parse Validation Result node falls back to a `confidenceScore: 50` with `overallAssessment: "REVIEW_NEEDED"`, ensuring unparseable responses get flagged for human review.
- **AI Agent failure:** The Run Validation node has `retryOnFail: true` with a 5-second wait between retries.
- **Complete sub-workflow failure:** If the sub-workflow crashes entirely, the **Validation Retry Queue** workflow (`workflows/Validation Retry Queue.json`) captures the error via n8n's Error Trigger, logs it to the "Retry Queue" sheet tab, and retries every 30 minutes (up to 3 attempts). After 3 failures, the status is set to `FAILED_PERMANENT` and an escalation email is sent to `notifyEmail`.

---

## Supporting Workflows

### Review Response Handler

**File:** `workflows/Review Response Handler.json`

Serves a web form via webhook that allows reviewers to record their disposition directly from the review alert email. Updates the audit log row (matched on Execution ID) with: Reviewed By, Review Date, Disposition, Review Notes.

**Webhook URL:** `https://simpledotbiz.app.n8n.cloud/webhook/review-response`

### Validation Retry Queue

**File:** `workflows/Validation Retry Queue.json`

Two-flow workflow:
1. **Error capture:** Triggered by the sub-workflow's Error Workflow setting. Logs failed executions to the "Retry Queue" sheet tab and sends a failure alert email.
2. **Scheduled retry:** Runs every 30 minutes, reads PENDING items, retries the sub-workflow, and updates status (RESOLVED or FAILED_PERMANENT after 3 attempts).

**Setup:** In the n8n UI, set the AI Validation Sub-Workflow's Error Workflow setting to point to this workflow.
