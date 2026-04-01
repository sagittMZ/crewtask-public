# AI Orchestration Pipeline

How natural language becomes a structured family task.

---

## Overview

The AI pipeline is the primary input method for CrewTask. A user speaks or types something like:

> "Dentist for both kids next Tuesday, remind dad the day before, take them from school"

The pipeline returns a structured task: title, assignees, due date, location hint, reminder schedule, checklist items.

This is not a single prompt → single response pattern. It is a custom multi-step pipeline where each stage has a defined responsibility.

---

## Pipeline Architecture

The pipeline adapts based on input type (`inputSource` field: `text`, `voice`, or `image`).

**Text input** (most common):
```
User types input
        │
        ▼
  [S2 - Extract]             ← text → structured task JSON (title, date, who, recurring, etc.)
        │                       sets is_ambiguous flag if input unclear
        ▼
  [S3 - Disambiguate]        ← conditional: only if S2 set is_ambiguous=true
        │                       generates a clarifying question shown to the user
        ▼
  [S4 - Conflict Alert]      ← conditional: only if conflicts detected against existing tasks
        │
        ▼
  Structured task (JSON)
```

**Voice input:**
```
User speaks
        │
        ▼
  [S1 - Normalize]           ← clean ASR artifacts: stuttering, mixed languages, phonetic errors
        │
        ▼
  [S2 - Extract] → [S3] → [S4] (same as text path above)
```

**Image input:**
```
User attaches photo (receipt, whiteboard, note)
        │
        ▼
  [Gemini Vision]            ← extracts task text from image
        │
        ▼
  [S2 - Extract] → [S3] → [S4] (same as text path above)
```

S1 runs only for voice input - text input goes straight to S2. This reduces API calls by ~66% for the common text case compared to earlier versions where S1 and a separate title-generation step ran unconditionally.

---

## Model Routing

Each pipeline step has a primary model and a Gemini-level fallback. If the primary returns a rate-limit or server error, the fallback is called automatically.

If all Gemini models fail (daily quota exhausted or provider outage), the pipeline falls back to **Groq (llama-3.3-70b-versatile)** as an emergency provider. This keeps the feature available even during Gemini downtime.

```
Step request
    │
    ▼
Primary Gemini model
    │ (429 / 5xx)
    ▼
Fallback Gemini model
    │ (also fails)
    ▼
Groq emergency fallback
    │ (also fails)
    ▼
Error returned to user
```

Current model assignments:

| Step | Primary | Fallback |
|------|---------|----------|
| S1 Normalize (voice) | gemini-2.5-flash-lite | gemini-2.5-flash |
| S2 Extract | gemini-2.5-flash | gemini-2.5-flash-lite |
| S3 Disambiguate | gemini-2.5-flash | gemini-2.5-flash-lite |
| S4 Conflict Alert | gemini-2.5-flash-lite | gemini-2.5-flash |

---

## Voice Input: Three-Path STT

Speech-to-text follows a fallback chain:

1. **Native STT** (Android/iOS system) - lowest latency
2. **Web Speech API** (Chrome/browsers) - web fallback
3. **Audio transcription via Edge Function** - for devices where neither option is available; records audio client-side (MediaRecorder API), sends to Edge Function, returns transcript

Voice input features:
- Auto-restart on silence (Voice Activity Detection, 7-second threshold)
- Hard cap at 90 seconds
- Append mode - multiple voice segments concatenate into one draft
- Draft auto-save every 15 minutes

---

## Observability

Every pipeline execution generates a trace stored server-side:
- Per-step latency
- Model tier used per step (primary, fallback, or groq-emergency)
- Output preview per step
- Error and fallback events
- A `run_id` returned to the client for support queries

Trace data is collected from day one. A dashboard to browse and analyze traces is planned - currently accessible via the database for debugging specific issues.

---

## Eval Suite

The pipeline has an automated evaluation suite: 40 input/expected pairs across 8 categories (basic, time extraction, recurring tasks, date, priority, all-day, multilingual, complex). Accuracy gates: overall >= 85%, critical categories (recurring, time) >= 80%.

The suite is triggered manually via `workflow_dispatch` in CI. Scheduled weekly runs are disabled until a paid AI API plan is in place - running 40 cases hits free-tier rate limits.

---

## Security

The pipeline runs entirely server-side in Edge Functions. AI model credentials are never exposed to the client. User data sent to the model is scoped to the requesting user's context - no cross-user data in prompts.
