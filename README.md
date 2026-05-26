# n8n AI Validation System

> **A production-grade, HIPAA-compliant AI hallucination detection and audit system built on n8n.**  
> Designed for a disability law firm processing sensitive client data through multiple AI models.

---

## Overview

This project implements a **reusable AI validation sub-workflow** that acts as a second-pass auditor for any AI-generated output in an n8n automation stack. It was built to solve a real problem: AI models hallucinate — and in a legal/healthcare context, a hallucinated attorney name, fabricated statistic, or missed call flag can have serious consequences.

The system uses **GPT-4.1-mini at temperature=0** to validate the output of other AI models (Claude, GPT-4.1-mini, Gemini) against the original source material. Every result — pass or fail — is logged to a Google Sheets audit trail. Suspicious outputs trigger an HTML email alert with one-click review buttons for human reviewers.

**Key design principle:** This is an audit and notification layer, not a gate. The parent workflow always continues with its original AI output. Validation is non-blocking by design.

---

## Architecture

```
Parent Workflow (Claude / GPT / Gemini produces AI output)
        │
        ▼
AI Validation Sub-Workflow
  ├── Prepare Context (normalize inputs, validate required fields)
  ├── Build Prompt (task-type-specific source-enforcement prompt)
  ├── GPT-4.1-mini @ temp=0 (second-pass validator)
  ├── Parse Result (extract structured JSON from AI response)
  ├── Apply Strictness Threshold (3-trigger decision logic)
  ├── Write Audit Log → Google Sheets (every result, pass or fail)
  └── Route Decision
        ├── FLAGGED → Format Alert → Send Email → Return Results
        └── PASSED  ──────────────────────────→ Return Results
        
Parent Workflow continues either way ↑
```

---

## Key Features

### 🔍 Three Validation Modes

| Mode | Use Case | What It Checks |
|------|----------|----------------|
| **EVALUATION** | Call center QA, classification tasks | False positives, false negatives, evidence strength |
| **GENERATION** | Legal content, SEO articles, video scripts | Claim-by-claim source tracing (VERIFIED / PLAUSIBLE / UNSOURCED) |
| **CUSTOM** | Any workflow with unique rules | Free-form `customValidationInstructions` as validation criteria |

### ⚖️ Three-Trigger Review Logic

An email alert fires if **any** of these conditions are true:

1. **Score below threshold** — confidence score < `strictnessThreshold` (configurable per workflow)
2. **HIGH-severity issue** — any flagged issue with severity = HIGH (e.g., hallucinated attorney name, fabricated statute)
3. **AI says REVIEW_NEEDED or FAIL** — the validator's own judgment directly triggers the alert

No alert is sent **only when all three** are clear: score passes, zero HIGH issues, AI says PASS.

### 📊 Configurable Strictness Threshold

One parameter controls sensitivity per workflow. Designed to be tuned over time as trust builds:

| Phase | Threshold | Purpose |
|-------|-----------|---------|
| Initial release (week 1–2) | 85–95 | Build a baseline; nearly everything flagged |
| Building trust (week 3–6) | 65–80 | Only medium/low-confidence outputs flagged |
| Trusted operation (month 2+) | 40–60 | Only genuinely suspicious outputs flagged |
| Mature/stable (month 3+) | 20–35 | Only clearly wrong outputs flagged |

### 🔒 HIPAA-Compliant Design

- **Zero PHI in email alerts** — review emails contain only a link to the n8n execution log (accessible only to authorized users), never raw AI output or client data
- **Audit log treated as ePHI** — Google Sheets audit log stored in BAA-covered Google Workspace with restricted access
- **BAA-aware architecture** — all AI providers (OpenAI, Anthropic, Google, Deepgram) operate under executed Business Associate Agreements
- **Minimum necessary standard** — only the data required for validation is passed to the validator

---

## Workflow Inventory

| Workflow | Description |
|----------|-------------|
| `AI Validation Sub-Workflow` | Core reusable validator — called by all parent workflows |
| `AI Validation Test Harness` | 9-scenario test suite (6 regular + 3 edge cases) for validating the validator itself |
| `Review Response Handler` | Webhook-based web form that lets reviewers log their disposition (Confirmed OK / Corrected / Escalated) directly from the email alert |
| `Validation Retry Queue` | Error capture + scheduled retry system — logs failed validations, retries every 30 min (up to 3 attempts), escalates on permanent failure |
| `KOBE - Hogan Smith Automation` | SEO content generation for disability law FAQ pages (GPT-4.1-mini) |
| `KOBE - Hogan Smith Automation (with validation)` | Same workflow with AI Validation Sub-Workflow integrated |
| `HSLGuestServicesTeam Call Transcribe, Flag, Forward` | Deepgram transcription → Claude Haiku call evaluation → email routing |
| `HSLGuestServicesTeam Call Transcribe, Flag, Forward (with validation)` | Same workflow with AI Validation Sub-Workflow integrated |
| `Veo3 Hogan Smith Social Media Prompt` | Gemini 2.5 Pro video script generation for social media |
| `Pre-hearing Memo Writer` | AI-assisted legal memo drafting |
| `Receive Document, check DDE, compile docs and send dashboard` | Document intake, DDE compliance check, and dashboard delivery |

---

## Tech Stack

- **Workflow Automation:** [n8n](https://n8n.io) (self-hosted on n8n Cloud)
- **AI Models:** OpenAI GPT-4.1-mini, Anthropic Claude Haiku, Google Gemini 2.5 Pro
- **Transcription:** Deepgram
- **Audit Storage:** Google Sheets (BAA-covered Google Workspace)
- **Notifications:** Gmail (OAuth2)
- **Language:** JavaScript (n8n Code nodes)

---

## Sub-Workflow Input Parameters

```json
{
  "workflowName": "KOBE - Hogan Smith Automation",
  "aiTaskType": "GENERATION",
  "originalInput": "The raw data fed to the AI (transcript, question, case info)",
  "aiOutput": "The AI's actual output being validated",
  "notifyEmail": "reviewer@example.com",
  "aiModel": "gpt-4.1-mini",
  "domainContext": "disability law",
  "strictnessThreshold": 75,
  "customValidationInstructions": ""
}
```

---

## Sample Return Payload

```json
{
  "passed": false,
  "needsHumanReview": true,
  "confidenceScore": 42,
  "overallAssessment": "REVIEW_NEEDED",
  "reviewLevel": "HIGH",
  "decisionSummary": "FLAGGED: Score 42/75 threshold. 1 high-severity issue(s). Review level: HIGH.",
  "flaggedIssues": [
    {
      "issue": "Statistic '$12,000 average SSDI benefit' not found in source material",
      "severity": "HIGH",
      "recommendation": "Remove or source this statistic before publishing"
    }
  ],
  "sourceCitations": [
    {
      "claim": "$12,000 average annual SSDI benefit",
      "source": "NOT SOURCED",
      "status": "UNSOURCED"
    }
  ],
  "unsourcedClaimCount": 1,
  "totalClaimsChecked": 8,
  "originalAiOutput": "<original AI output passed through unchanged>"
}
```

---

## Error Handling

| Failure Mode | Behavior |
|---|---|
| Missing required fields | Returns `passed: true`, `overallAssessment: "ERROR"` — parent workflow never blocked |
| AI response parse failure | Defaults to `confidenceScore: 50`, `overallAssessment: "REVIEW_NEEDED"` — ensures failures get flagged |
| AI Agent failure | `retryOnFail: true` with 5-second wait between retries |
| Complete sub-workflow crash | Captured by Validation Retry Queue — logged, retried every 30 min, escalated after 3 failures |

---

## Audit Log

Every validation appends a row to the **AI Validation Audit Log** Google Sheet:

`Timestamp | Workflow Name | Task Type | AI Model | Confidence Score | Overall Assessment | Strictness Threshold | Passed | Flagged Issues Count | High Severity Count | Unsourced Claims | Total Claims Checked | Reasoning | Execution ID`

When a reviewer submits their response via the review form, the same row is updated with:

`Reviewed By | Review Date | Disposition | Review Notes`

---

## Background

This system was built in response to a real operational need: a disability law firm using AI to process client calls, generate legal FAQ content, and create social media video scripts needed a systematic way to catch AI errors before they caused compliance or reputational issues.

The original brief: *"Franco R is tasked with building an AI confidence indicator and a recheck system to systematically validate AI outputs against hallucinations in areas of legal risk."*

The result is a production-deployed, HIPAA-aware validation layer that runs silently behind every AI workflow — logging everything, flagging what needs human eyes, and never blocking the business process.

---

## License

MIT
