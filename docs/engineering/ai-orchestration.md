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

```
User input (text or voice)
        │
        ▼
  [S1 - Normalize]          ← clean input: typos, mixed languages, voice artifacts
        │
        ▼
  [S2 - Extract] ──────┐    ← text → structured task schema (who, when, recurring?)
                        │         sets is_ambiguous flag if input unclear
  [S7 - Title]  ────────┘    ← clean task title (runs in parallel with S2)
        │
        ▼
  [S3 - Disambiguate]        ← conditional: only runs if S2 set is_ambiguous=true
        │                       generates a clarifying question shown to the user
        ▼
  [S4 - Conflict Detection]  ← check against existing tasks/schedule
        │
        ▼
  Structured task (JSON)
```

S2 and S7 run in parallel to minimize latency. Each step runs as a separate call within the Edge Function pipeline.

---

## Model Routing

Currently the pipeline uses one AI provider. Each step has a primary model and a fallback - if the primary returns an error (rate limit, timeout), the fallback is called automatically. The routing layer is designed to support multiple providers per step; multi-provider routing is planned as the product scales.

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
- Model tier used per step (primary or fallback)
- Output preview per step
- Error and fallback events
- A `run_id` returned to the client for support queries

Trace data is collected from day one. A dashboard to browse and analyze traces is planned - currently accessible via the database for debugging specific issues.

---

## Eval Suite

The pipeline has an automated evaluation suite: 40 input/expected pairs across 8 categories (basic, time extraction, recurring tasks, date, priority, all-day, multilingual, complex). Runs weekly on CI with accuracy gates: overall >= 85%, critical categories (recurring, time) >= 80%.

This means prompt changes are validated against known-good outputs before reaching production.

---

## Security

The pipeline runs entirely server-side in Edge Functions. AI model credentials are never exposed to the client. User data sent to the model is scoped to the requesting user's context - no cross-user data in prompts.
