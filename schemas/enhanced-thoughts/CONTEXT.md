# CONTEXT.md — Enhanced Thoughts

## Purpose

Extends the core `thoughts` table with structured classification columns and installs three utility RPCs that power full-text search, aggregate brain statistics, and thought-connection discovery. This schema is the foundation for any feature that filters, ranks, or relates thoughts by type, importance, or sensitivity.

## Responsibility Boundaries

- **Owns**: The six added columns (`type`, `sensitivity_tier`, `importance`, `quality_score`, `source_type`, `enriched`), their indexes, and the three RPCs (`search_thoughts_text`, `brain_stats_aggregate`, `get_thought_connections`)
- **Delegates to**: Callers to populate `type` and `source_type` correctly; the `metadata` JSONB column (core table) is the authoritative source during backfill
- **Does not handle**: Embedding/vector search (that remains the core table's concern), row-level security policy enforcement (RLS is separate), or writes to the `enriched` flag beyond schema definition

## Key Concepts

- **sensitivity_tier**: A three-value access classification (`standard`, `restricted`, and implicitly anything else). The `restricted` tier is used as an exclusion signal in all three RPCs via the `p_exclude_restricted` parameter.
- **importance / quality_score**: Numeric signals (1–10 smallint and 0–100 numeric respectively) that are blended into the `search_thoughts_text` rank formula alongside text relevance, so higher-importance or higher-quality thoughts surface above lower-scoring matches even with equal text relevance.
- **Two-phase full-text search**: `search_thoughts_text` runs a GIN-indexed `tsvector` match first (up to 2 000 hits) and only falls back to an `ILIKE` scan when the tsvector phase returns fewer results than `p_limit + p_offset`. This means ILIKE is only hit when the GIN index is insufficient, keeping the common case fast.
- **Rank formula**: `ts_rank_cd(...) OR 0.35 (ILIKE floor) + importance/20 + quality_score/500`. ILIKE hits receive a fixed floor rank of 0.35 so they do not outrank genuine tsvector hits.
- **Thought connections**: `get_thought_connections` derives relatedness purely from shared `metadata→topics` and `metadata→people` array values — no vector similarity is involved. Returns early (empty set) if the source thought has neither topics nor people.
- **Idempotent SQL**: All DDL uses `ADD COLUMN IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS`; RPCs use `CREATE OR REPLACE FUNCTION`. The backfill `UPDATE` statements are guarded by `WHERE ... IS NULL` so re-running is safe.

## Non-Obvious Details

- The `websearch_to_tsquery('simple', ...)` parser (not `plainto_tsquery`) is intentional: it enables boolean operators (`"quoted phrases"`, `-NOT`, `OR`) that end users might type naturally.
- `brain_stats_aggregate` counts total thoughts **all-time** but scopes `top_types` and `top_topics` to the `p_since_days` window. These two counts will diverge when the window is short.
- The backfill only promotes a fixed allowlist of `type` values (`idea`, `task`, `person_note`, `reference`, `decision`, `lesson`, `meeting`, `journal`) from `metadata->>'type'`; any other value is left as NULL even if present in metadata.
- `NOTIFY pgrst, 'reload schema'` at the end forces PostgREST to pick up the new columns and functions immediately without a container restart.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, idempotent schema migration)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, idempotent schema migration)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, sensitivity_tier)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, sensitivity_tier, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, sensitivity_tier)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, sensitivity_tier)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, sensitivity_tier)
- **[docs](../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, sensitivity_tier)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, sensitivity_tier)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, sensitivity_tier)
- **[integrations](../../integrations/CONTEXT.md)** — Provides search_thoughts_text, brain_stats_aggregate, get_thought_connections consumed by this module
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Provides search_thoughts_text, brain_stats_aggregate, get_thought_connections consumed by this module
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), importance/quality_score ranking signals)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, sensitivity_tier)
- **[integrations/kubernetes-deployment/k8s](../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Supabase schema parity, hostPath volume, idempotent schema migration)
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, importance/quality_score ranking signals)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Source Confidence, importance/quality_score ranking signals)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Sync log, idempotent schema migration)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, importance/quality_score ranking signals)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Sync log, Two-layer dedup, idempotent schema migration)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, metadata-overlap thought connections)
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, idempotent schema migration)
- **[recipes/google-activity-import](../../recipes/google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, idempotent schema migration)
- **[recipes/grok-export-import](../../recipes/grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, idempotent schema migration)
- **[recipes/instagram-import](../../recipes/instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), idempotent schema migration)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, idempotent schema migration)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, idempotent schema migration)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Session-scoped deduplication, idempotent schema migration)
- **[recipes/local-ollama-embeddings](../../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, two-phase GIN+ILIKE full-text search)
- **[recipes/ob-graph](../../recipes/ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, metadata-overlap thought connections, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), idempotent schema migration)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Local sync log deduplication, idempotent schema migration)
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, idempotent schema migration)
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, idempotent schema migration)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), sensitivity_tier)
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, idempotent schema migration)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, sensitivity_tier)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (match_thoughts, two-phase GIN+ILIKE full-text search)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (derived_from edges, metadata-overlap thought connections)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (importance/quality_score ranking signals, source_confidence)
- **[recipes/x-twitter-import](../../recipes/x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, idempotent schema migration)
- **[schemas](../CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier, sensitivity_tier access filtering)
- **[schemas/entity-extraction](../entity-extraction/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, metadata-overlap thought connections)
- **[schemas/typed-reasoning-edges](../typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), metadata-overlap thought connections, support_count evidence accumulation)
- **[server](../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier, x-brain-key access key auth)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, idempotent schema migration)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, importance/quality_score ranking signals)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality flags, importance/quality_score ranking signals)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (importance/quality_score ranking signals, quality flags)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Starter universe bootstrap, idempotent schema migration)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (importance/quality_score ranking signals, source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, importance/quality_score ranking signals)
