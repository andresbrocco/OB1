# CONTEXT.md — Schemas

## Purpose

Database table extensions for the Open Brain `thoughts` table and its associated knowledge graph. Each sub-folder is a standalone, idempotent SQL migration that community contributors can apply to extend the core Supabase database without modifying the protected `thoughts` table structure.

## Responsibility Boundaries

- **Owns**: All community-contributed schema additions — new tables, columns, indexes, RPCs, triggers, and RLS policies that extend the Open Brain data model.
- **Delegates to**: `docs/01-getting-started.md` for the core `thoughts` table definition; `recipes/` and `integrations/` for the workers and edge functions that consume these tables.
- **Does not handle**: The core `thoughts` table columns (`id`, `content`, `embedding`, `created_at`, etc.) — those are off-limits to modification. Application-layer logic also lives elsewhere (recipes, skills, integrations).

## Key Concepts

- **Two distinct edge tables**: `edges` (entity-to-entity, from `entity-extraction`) and `thought_edges` (thought-to-thought, from `typed-reasoning-edges`) serve different graph layers. Confusing them is a common pitfall: `edges.from_entity_id` references `entities.id` (BIGINT), while `thought_edges.from_thought_id` references `thoughts.id` (UUID).
- **Reasoning relation vocabulary** (`thought_edges`): `supports`, `contradicts`, `evolved_into`, `supersedes`, `depends_on`, `related_to`. Only these six values are accepted via a CHECK constraint.
- **Temporal validity**: Both edge tables carry `valid_from`, `valid_until`, and `decay_weight` columns. `NULL valid_until` means "still current"; `decay_weight` is a 0–1 float recalculated by a decay job to deprioritize stale edges in graph traversal.
- **Entity extraction queue**: A trigger on `thoughts` (`trg_queue_entity_extraction`) auto-enqueues rows for async processing. It skips rows where `metadata->>'generated_by'` is set (system artifacts) and is a no-op if `content_fingerprint` has not changed.
- **sensitivity_tier** (`enhanced-thoughts`): A column controlling access-tier filtering in RPCs. The value `'restricted'` is used as an exclusion filter in `brain_stats_aggregate` and `get_thought_connections`.

## Non-Obvious Details

- **`entity-extraction` has a hard prerequisite**: The migration checks for `thoughts.content_fingerprint` at runtime and raises an exception with an actionable message if the column is absent. Applying it before completing `docs/01-getting-started.md` Step 2.6 will fail loudly.
- **`typed-reasoning-edges` requires `entity-extraction` to be applied first**: It adds temporal validity columns to the `edges` table shipped by that schema. Applying out of order will fail with a prerequisite check exception.
- **RLS posture diverges between schemas**: `entity-extraction` grants `authenticated` SELECT access to the new tables (scaffolded for future multi-tenancy). `typed-reasoning-edges` intentionally restricts `thought_edges` to `service_role` only, matching the posture of `public.thoughts`, because reasoning edges expose derived private-thought relationships.
- **Backfill steps are opt-in**: Both `entity-extraction` and `workflow-status` include commented-out or guarded `UPDATE`/`INSERT` backfill statements. The trigger only fires on future inserts/updates; pre-existing rows need the backfill run manually once.
- **`search_thoughts_text` uses a two-phase search**: GIN tsvector index first (fast, up to 2000 hits); ILIKE fallback only runs when tsvector returns fewer results than `p_limit + p_offset`. The ranking formula blends `ts_rank_cd`, ILIKE presence (fixed 0.35), and normalized `importance` / `quality_score` boosts.
- **`thought_edges_upsert` NULL semantics**: On conflict, `valid_until = NULL` wins over any concrete date (NULL means "still current"), and `valid_from` takes the earlier non-NULL bound. This matches the typed-edge-classifier contract but differs from standard GREATEST/LEAST NULL behavior.

## Related Modules

- **[dashboards](../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier access filtering, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, sensitivity_tier access filtering)
- **[dashboards/open-brain-dashboard-next](../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, sensitivity_tier access filtering, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, sensitivity_tier access filtering)
- **[dashboards/open-brain-dashboard-next/components](../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, sensitivity_tier access filtering)
- **[dashboards/open-brain-dashboard-next/lib](../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, sensitivity_tier access filtering, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src/routes](../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, sensitivity_tier access filtering)
- **[docs](../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, sensitivity_tier access filtering)
- **[extensions/family-calendar](../extensions/family-calendar/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Temporal validity and decay_weight, recurring vs. one-time activities in a unified table)
- **[extensions/home-maintenance](../extensions/home-maintenance/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Temporal validity and decay_weight, frequency_days=NULL for one-time tasks, trigger-driven next_due recalculation)
- **[extensions/household-knowledge](../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, sensitivity_tier access filtering)
- **[extensions/meal-planning](../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, sensitivity_tier access filtering)
- **[integrations](../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, ExtractionCostCapError, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, _enrichment_status)
- **[integrations/kubernetes-deployment](../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, sensitivity_tier access filtering)
- **[integrations/kubernetes-deployment/k8s](../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Two-phase full-text search (GIN tsvector + ILIKE fallback), match_thoughts RPC equivalent)
- **[recipes](../recipes/CONTEXT.md)** — Provides search_thoughts_text, brain_stats_aggregate, get_thought_connections, ... consumed by this module
- **[recipes/bring-your-own-context](../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Two-Prompt Extraction Sequence)
- **[recipes/claudeception](../recipes/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Extraction)
- **[recipes/entity-wiki](../recipes/entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), co_occurs_with exclusion)
- **[recipes/life-engine](../recipes/life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, External before internal enrichment)
- **[recipes/live-retrieval](../recipes/live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[recipes/local-ollama-embeddings](../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[recipes/ob-graph](../recipes/ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/repo-learning-coach/src](../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughts, Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[recipes/repo-learning-coach/src/lib](../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughtSummary, Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[recipes/schema-aware-routing](../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Pending person confirmation, Three-pass person resolution)
- **[recipes/thought-enrichment](../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), sensitivity_tier access filtering)
- **[recipes/typed-edge-classifier](../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to))
- **[recipes/vercel-neon-telegram/src](../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier access filtering, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, sensitivity_tier access filtering)
- **[recipes/vercel-neon-telegram/src/lib](../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Two-phase full-text search (GIN tsvector + ILIKE fallback), match_thoughts)
- **[recipes/wiki-synthesis](../recipes/wiki-synthesis/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, Temporal validity and decay_weight)
- **[recipes/wiki-synthesis/scripts](../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), derived_from edges)
- **[schemas/enhanced-thoughts](enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier, sensitivity_tier access filtering)
- **[schemas/entity-extraction](entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Entity extraction queue with auto-trigger, Thought-entity mention role and evidence)
- **[schemas/typed-reasoning-edges](typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), Two-tier edge model (entity edges vs thought edges), support_count evidence accumulation)
- **[server](../server/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier access filtering, x-brain-key access key auth)
- **[skills/claudeception](../skills/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Extraction threshold and quality gates)
- **[skills/panning-for-gold](../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Thread extraction)
- **[skills/weekly-signal-diff](../skills/weekly-signal-diff/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, Two-phase full-text search (GIN tsvector + ILIKE fallback))
