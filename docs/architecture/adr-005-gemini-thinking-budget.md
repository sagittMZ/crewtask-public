# ADR-005: Gemini Thinking Budget — Disabled for Extraction, Explicit for Reasoning

**Date:** 2026-05-15 | **Status:** Accepted

---

## Context

Gemini 2.5 Flash enables a "thinking" mode by default (budget: 8192 tokens). When active, the model generates internal reasoning tokens before producing its output. These tokens count toward the output and are billed at a higher rate than standard output.

The parse-task pipeline uses Gemini 2.5 Flash for two steps: S2 (structured extraction) and S3 (disambiguation). Neither step was specifying `thinkingConfig` explicitly, which meant thinking was active by default — without a deliberate decision to enable it.

A review triggered by the Gemini 2.5 series upgrade (May 2026) surfaced this: extraction tasks do not benefit from thinking. The reasoning process adds latency and cost without improving output quality on structured, schema-driven tasks.

---

## Decision

**1. Disable thinking for all current pipeline steps.**

All extraction and normalization steps (`S1`, `S2`, `S3`, `S4`, `S7`) now set `thinkingConfig: { thinkingBudget: 0 }` explicitly. This applies to both `task-standard` (Flash) and `task-lite` (Flash-Lite, which has no thinking capability anyway).

**2. Document the thinking budget in the model registry.**

Each model role in `_shared/models.ts` now carries a `thinkingBudget` field. The intent: any developer adding a new pipeline step must read the config and make a deliberate choice. The budget is not inherited implicitly from the model default.

**3. Reserve thinking for future reasoning tasks.**

Route optimization and schedule analysis involve genuine multi-step reasoning — the kind of problem where thinking tokens improve output quality. When those features ship, they will use `task-advanced` (Gemini 2.5 Pro) or a dedicated config with an explicit, bounded budget (e.g., 4096 tokens).

---

## Why Task Extraction Does Not Need Thinking

The pipeline steps are deterministic and schema-driven:

| Step | Task | Why thinking doesn't help |
|------|------|--------------------------|
| S1 Normalize | Clean ASR artifacts | Pattern replacement, no reasoning required |
| S2 Extract | Parse text → JSON schema | Output shape is fixed; model follows a template |
| S3 Disambiguate | Generate a clarifying question | One clear question per gap; no search space |
| S4 Conflict Alert | Explain a scheduling conflict | Describing a detected fact |
| S7 Title | 3-5 word title | Summarization, not reasoning |

Thinking helps when the solution space is large or the problem requires backtracking. None of these steps have that property.

---

## Where Thinking Will Be Used

Planned features that justify thinking:

- **Route optimization** — finding efficient task sequences across locations is a combinatorial problem
- **Schedule analysis** — identifying patterns across a family's history requires multi-step inference

These will use an explicit, bounded budget rather than the model default. Budget is set per use case, not globally.

---

## Consequences

**Gained:**
- Consistent, predictable model behavior across pipeline steps
- Lower latency per pipeline run
- `thinkingBudget` is now a documented, explicit field in the model registry — no implicit defaults

**Lost / risk:**
- S3 (disambiguation) may produce slightly less creative clarifying questions on edge cases. If regressions appear in eval, a small budget (`512`) can be added to S3 specifically.

**Not changed:**
- Flash-Lite (`task-lite`) has no thinking capability; this ADR formalizes what was already true.
- Embedding model is unaffected.

---

## Related

- ADR-004: Android App Links and Deep Links
- [`docs/engineering/ai-orchestration.md`](../engineering/ai-orchestration.md) — pipeline step details and model assignments
