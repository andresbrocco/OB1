# CONTEXT.md — _shared

## Purpose

Shared constants, type definitions, and utility functions consumed by all edge functions in the entity-extraction-worker integration. Centralizes the full thought ingestion pipeline: structured-capture parsing, multi-provider LLM metadata extraction, sensitivity detection, embedding generation, content fingerprinting, and final payload assembly.

## Responsibility Boundaries

- **Owns**: All logic for transforming raw text into a `PreparedPayload` ready to insert into Supabase — including provider selection, retry/fallback, metadata sanitization, sensitivity classification, and precedence resolution for all fields.
- **Delegates to**: External HTTP APIs (OpenRouter, OpenAI, Anthropic) for embeddings and LLM classification; callers supply the Supabase client for table existence checks.
- **Does not handle**: Database writes, MCP protocol framing, route handling, or authentication. Those remain in the individual edge function entry points.

## Key Concepts

- **`prepareThoughtPayload`**: The canonical ingest pipeline (14 steps). All capture paths (MCP `capture_thought`, REST `/capture`, smart-ingest) must funnel through this function. Override precedence is: structured-capture hint > caller-supplied metadata > LLM-extracted metadata > defaults.
- **Structured capture format**: Inline syntax `[type] [topic] body text + next step` parsed by `parseStructuredCapture`. When matched, type/topic hints bypass LLM classification for those fields and a fixed confidence of `0.82` is applied.
- **Importance scale (0–6)**: Importance 6 is exclusively user-flagged and must never be auto-assigned by LLM or pipeline code. The classifier prompt and `sanitizeMetadata` both cap auto-assignment at 5.
- **Sensitivity tiers**: `standard → personal → restricted`. Escalation-only: once detected or overridden to a higher tier, it cannot be downgraded. Unrecognized caller-supplied tier values normalize to `"personal"` as a safe default.
- **Provider priority**: OpenRouter is primary for embeddings and classification; OpenAI is secondary; Anthropic is tertiary. This is the reverse of the upstream ExoCortex project, from which this code was ported.
- **`_enrichment_status`**: `"complete"` | `"fallback"` | `"skipped"` — propagated into metadata so downstream queries can filter by enrichment quality.

## Non-Obvious Details

- `sanitizeMetadata` clamps LLM-assigned importance to `0–5` (never 6), even if the model returns 6 in its JSON response.
- `resolveSensitivityTier` maps any unrecognized string to `"personal"` rather than `"standard"`, intentionally erring toward privacy.
- `embedText` and `fetchOpenRouterMetadata` both read model names from environment variables at call time (`OPENROUTER_EMBEDDING_MODEL`, `OPENROUTER_CLASSIFIER_MODEL`, etc.), allowing per-deployment model overrides without code changes.
- `extractMetadata` retries the primary provider only on transient errors (network failures, 429, 5xx); it does not retry on parse errors or bad responses, and falls through to secondary/tertiary providers instead.
- `quality_score` defaults to `Math.round(confidence * 70 + 20)` when not caller-supplied, bounding quality between 20 and 90 based on classifier confidence.
- The `MAX_TAGS_PER_THOUGHT = 12` limit is enforced in `normalizeStringArray` (slice after dedup), not as a separate guard.

## Related Modules

- **[.github](../../../.github/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[.github/workflows](../../../.github/workflows/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Two-stage review pipeline)
- **[dashboards](../../../dashboards/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, Structured capture format, prepareThoughtPayload)
- **[dashboards/open-brain-dashboard-next](../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), quality audit (quality_score <= 29))
- **[dashboards/open-brain-dashboard-next/app/api](../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Importance scale (0-6, 6 is user-only))
- **[dashboards/open-brain-dashboard-next/components](../../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), Structured capture format, prepareThoughtPayload)
- **[dashboards/open-brain-dashboard-next/lib](../../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Structured capture format, prepareThoughtPayload)
- **[dashboards/open-brain-dashboard/src/routes](../../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, _enrichment_status)
- **[integrations](../../CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (_enrichment_status, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, _enrichment_status, entity_extraction_queue)
- **[recipes/adaptive-capture-classification](../../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Importance scale (0-6, 6 is user-only), Per-type thresholds)
- **[recipes/bring-your-own-context](../../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Two-Prompt Extraction Sequence, _enrichment_status)
- **[recipes/chatgpt-conversation-import](../../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[recipes/claudeception](../../../recipes/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, _enrichment_status)
- **[recipes/email-history-import](../../../recipes/email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Ingestion modes, Structured capture format, prepareThoughtPayload)
- **[recipes/google-activity-import](../../../recipes/google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, Thought prefix format on insert, prepareThoughtPayload)
- **[recipes/instagram-import](../../../recipes/instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, prepareThoughtPayload, upsert_thought RPC)
- **[recipes/life-engine](../../../recipes/life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, _enrichment_status)
- **[recipes/obsidian-vault-import](../../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Structured capture format, prepareThoughtPayload)
- **[recipes/perplexity-conversation-import](../../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Selective LLM summarization)
- **[recipes/schema-aware-routing](../../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Three-pass person resolution, _enrichment_status)
- **[recipes/thought-enrichment](../../../recipes/thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, _enrichment_status)
- **[recipes/typed-edge-classifier](../../../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[recipes/vercel-neon-telegram](../../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, Structured capture format, prepareThoughtPayload)
- **[recipes/vercel-neon-telegram/src](../../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, captureThought pipeline, prepareThoughtPayload)
- **[recipes/vercel-neon-telegram/src/lib](../../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, captureThought pipeline, prepareThoughtPayload)
- **[recipes/wiki-synthesis](../../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Synthesizer catalogue)
- **[recipes/wiki-synthesis/scripts](../../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Synthesizer catalogue (plugin map))
- **[recipes/work-operating-model-activation](../../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), source_confidence)
- **[schemas](../../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, _enrichment_status)
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), importance/quality_score ranking signals)
- **[schemas/entity-extraction](../../../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, _enrichment_status)
- **[server](../../../server/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, Two-step capture (upsert + embedding patch), prepareThoughtPayload, upsert_thought RPC (deduplication-aware insert))
- **[skills/claudeception](../../../skills/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, _enrichment_status)
- **[skills/financial-model-review](../../../skills/financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, Importance scale (0-6, 6 is user-only))
- **[skills/heavy-file-ingestion](../../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[skills/heavy-file-ingestion/scripts](../../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), quality flags)
- **[skills/n-agentic-harnesses](../../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Thread extraction, _enrichment_status)
- **[skills/work-operating-model](../../../skills/work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Importance scale (0-6, 6 is user-only))
