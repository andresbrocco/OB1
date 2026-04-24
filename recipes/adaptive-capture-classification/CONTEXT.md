# CONTEXT.md — Adaptive Capture Classification

## Purpose

Wraps OB1's capture flow with LLM-based type classification, confidence gating, and a per-type adaptive threshold learning loop. Rather than always auto-classifying or always prompting the user, it decides dynamically based on learned confidence thresholds that evolve from user feedback over time.

## Responsibility Boundaries

- **Owns**: LLM classification calls, confidence gating logic, per-type threshold management, outcome recording, and feedback-driven threshold adjustment
- **Delegates to**: Caller for user confirmation UI; OB1 MCP tool or Supabase `thoughts` table for the actual capture write; any OpenAI-compatible LLM gateway (OpenRouter by default)
- **Does not handle**: Spell correction execution (schema supports it via `correction_learnings`, but no runtime logic is included), A/B model comparison execution (schema table exists but calling code is not implemented), or the UI layer for prompting the user when confidence is low

## Key Concepts

- **Confidence gating**: The LLM self-reports a confidence score (0–10). The pipeline compares `confidence / 10` against a per-type learned threshold to decide whether to auto-classify or require user confirmation.
- **Per-type thresholds** (`capture_thresholds` table): Each OB1 capture type starts at 0.75. Accepted auto-classifications nudge it down (become more permissive); user corrections nudge it up (become more conservative). Clamped to [0.50, 0.95].
- **Consistency check**: For captures where the first LLM call returns confidence < 9, a second call is made. If the two calls disagree on `type`, the confidence score is multiplied by 0.6 as a penalty.
- **Two-phase pipeline**: `processCapture()` classifies and gates but does not write to OB1. `completeCapture()` finalises the capture after user confirmation, writes to OB1, records the outcome, and adjusts the threshold. This split allows the caller to interpose a confirmation UX.
- **`hintType` override**: If the caller passes a `hintType`, the classifier is instructed to use that type and the confidence is forced to 10, bypassing the gate entirely.

## Non-Obvious Details

- The `outcomeId` returned by `processCapture()` must be passed back to `completeCapture()` to close the feedback loop. If `completeCapture()` is never called, the outcome row stays with `user_accepted = null` and does not affect thresholds.
- `autoClassify: false` in the pipeline result means the caller is responsible for asking the user — the pipeline itself does not block or prompt. Callers that silently discard low-confidence results will corrupt the learning loop because `completeCapture()` will not be called.
- The `ab_comparisons` and `correction_learnings` tables are created by `schema.sql` but no runtime implementation exists in this recipe. They are scaffolding for extensions.
- Temperature is fixed at 0.1. Higher temperatures increase type field variability and make confidence scores harder to calibrate against the learned thresholds.
- The `OB1_TYPES` constant in `capture-with-gating.ts` must match the `type` column's valid values in the `thoughts` table. Adding types here without updating the table will cause insert failures at `writeToOB1()`.

## Related Modules

- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Two-phase pipeline)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, quality audit (quality_score <= 29))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Confidence gating, Per-type thresholds)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Importance scale (0-6, 6 is user-only), Per-type thresholds)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, Source Confidence)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Retrospective Mode, Two-phase pipeline)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Two-phase pipeline)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Self-improvement protocol, Two-phase pipeline)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, LLM classification prompt with importance/confidence calibration, Per-type thresholds)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Two-phase pipeline)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Phase toggles, Two-phase pipeline)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, source_confidence)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, importance/quality_score ranking signals)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Two-phase pipeline)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Retrospective mode, Two-phase pipeline)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Fatal issues vs. caution flags vs. acceptable simplifications, Per-type thresholds)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, Quality flags)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, quality flags)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Two-phase pipeline)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Five-principle evaluation, Per-type thresholds)
