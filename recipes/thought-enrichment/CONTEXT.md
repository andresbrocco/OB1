# CONTEXT.md — Thought Enrichment

## Purpose

Retroactively classifies existing thoughts in a Supabase Open Brain database by adding structured metadata (type, summary, topics, tags, people, action_items, confidence, importance, detected_source_type) via LLM calls, and by detecting and upgrading sensitivity tiers using regex pattern matching. Designed to run as a one-time or incremental backfill against an existing database of unclassified thoughts.

## Responsibility Boundaries

- **Owns**: LLM-based enrichment of the `thoughts` table (type, importance, metadata fields), regex-based sensitivity tier classification (`standard` / `personal` / `restricted`), and backfill of the `type` column from previously stored `metadata.type` values.
- **Delegates to**: Anthropic or OpenRouter APIs for classification inference; Supabase REST API for all reads and writes.
- **Does not handle**: Embedding generation, real-time capture, schema migrations, or thought creation.

## Key Concepts

- **Sensitivity tiers**: A three-level classification (`standard`, `personal`, `restricted`) applied to each thought based on regex detection of PII, health data, credentials, and financial details. Patterns are defined in `sensitivity-patterns.json` and split into `restricted` (credential/identity patterns that short-circuit on first match) and `personal` (health/financial patterns that accumulate reasons).
- **Enrichment versioning**: Enriched thoughts carry `metadata.enriched_version` (currently `1`) and `metadata.enriched_at` so that future schema changes can identify which thoughts need re-enrichment.
- **Backfill vs enrichment**: Three separate scripts serve distinct repair scenarios — `enrich-thoughts.mjs` calls an LLM to classify unenriched thoughts; `backfill-sensitivity.mjs` re-scans thoughts whose sensitivity tier is `standard`/null; `backfill-type.mjs` promotes the `type` column for rows where `type='reference'` but `metadata.type` already holds a valid non-reference value (a data-consistency repair for rows enriched before the `type` column was updated).
- **Resumable state**: `enrich-thoughts.mjs` tracks progress in `data/enrichment-state.json` (cursor, failure IDs, rate stats) to allow safe interruption and resumption. The state file is written atomically via a `.tmp` rename.

## Non-Obvious Details

- `backfill-sensitivity.mjs` imports sensitivity patterns from `./lib/sensitivity-patterns.mjs`, which is not present in this directory at the top level. This file must exist for the sensitivity backfill to run.
- The LLM classifier is instructed to return strict JSON with no markdown fences; the enrichment script strips fences defensively anyway, then validates and sanitizes all fields before writing — invalid `type` values fall back to `"reference"`, invalid `detected_source_type` values fall back to the existing `source_type` or `"generic_import"`.
- `enrich-thoughts.mjs` uses cursor-based pagination (`id > lastId`) for normal batches but falls back to offset-based pagination for the initial skip, avoiding expensive offset queries on large tables.
- Importance calibration in the prompt is intentionally conservative: the default is 3, with 5 reserved for life decisions and core health/financial data. This prevents score inflation across large corpora.
- The `--retry-failed` mode of `enrich-thoughts.mjs` re-processes only IDs stored in the local state file, not all unenriched rows, so transient API failures can be repaired without a full re-scan.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Two-stage review pipeline)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, Sensitivity tiers (standard/personal/restricted))
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Sensitivity tiers (standard/personal/restricted), Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, Sensitivity tiers (standard/personal/restricted))
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Sensitivity tiers (standard/personal/restricted))
- **[docs](../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, Sensitivity tiers (standard/personal/restricted))
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Sensitivity tiers (standard/personal/restricted))
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Sensitivity tiers (standard/personal/restricted))
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, ExtractionCostCapError, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, _enrichment_status)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Sensitivity tiers (standard/personal/restricted))
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, LLM classification prompt with importance/confidence calibration, Per-type thresholds)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Two-Prompt Extraction Sequence)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, LLM classification prompt with importance/confidence calibration)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Extraction)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Cursor-based resumable state)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (--redo flag, Cursor-based resumable state)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, External before internal enrichment)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Type backfill (metadata.type promotion))
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Pending person confirmation, Three-pass person resolution)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, LLM classification prompt with importance/confidence calibration)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Sensitivity tiers (standard/personal/restricted), Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (ThoughtMetadata, Type backfill (metadata.type promotion))
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumable state, Phase toggles)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Synthesizer catalogue)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Synthesizer catalogue (plugin map))
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (LLM classification prompt with importance/confidence calibration, source_confidence)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), sensitivity_tier)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Enrichment versioning, Thought-entity mention role and evidence)
- **[server](../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), x-brain-key access key auth)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Extraction threshold and quality gates)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, Type backfill (metadata.type promotion))
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, LLM classification prompt with importance/confidence calibration)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (LLM classification prompt with importance/confidence calibration, quality flags)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, Type backfill (metadata.type promotion))
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Thread extraction)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (LLM classification prompt with importance/confidence calibration, source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, LLM classification prompt with importance/confidence calibration)
