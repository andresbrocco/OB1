# CONTEXT.md — Typed Reasoning Edges

## Purpose

Extends the Open Brain knowledge graph with two capabilities: (1) a `thought_edges` table that captures typed semantic reasoning relations between `thoughts` rows, and (2) temporal validity columns (`valid_from`, `valid_until`, `decay_weight`) added to the existing entity-to-entity `edges` table from `schemas/entity-extraction`.

## Responsibility Boundaries

- **Owns**: The `thought_edges` table, its indexes, RLS policies, and the `thought_edges_upsert` RPC function; temporal validity columns on the pre-existing `edges` table.
- **Delegates to**: `recipes/typed-edge-classifier` for automated population of `thought_edges`; `schemas/entity-extraction` for the `edges` table that this schema augments.
- **Does not handle**: Entity-to-entity edges (those belong to `schemas/entity-extraction`); the `thoughts` table itself (defined in `docs/01-getting-started.md`); decay recalculation jobs (external to this schema).

## Key Concepts

- **Relation vocabulary**: The six allowed values — `supports`, `contradicts`, `evolved_into`, `supersedes`, `depends_on`, `related_to` — form a closed enum enforced by a CHECK constraint. `evolved_into` and `supersedes` are directional and distinct: `evolved_into` marks organic refinement while `supersedes` marks an explicit replacement (versions, decisions).
- **Temporal validity**: `valid_from`/`valid_until` NULLs have semantic meaning. NULL `valid_from` means "always/unknown"; NULL `valid_until` means "still current." The upsert function and decay indexes depend on these NULL semantics.
- **`decay_weight`**: A 0.0–1.0 float representing current relevance. Lower values cause an edge to rank lower in graph traversal. It is recalculated externally (by a classifier or a dedicated decay job) and is not auto-maintained by this schema.
- **`support_count`**: Incremented each time the same `(from_thought_id, to_thought_id, relation)` triple is re-classified. It accumulates evidence for an edge rather than overwriting it.
- **Dual-table scope**: This schema touches two tables that belong to different schemas. The `thought_edges` table is net-new here; the `edges` table modification is a guarded `ALTER TABLE ADD COLUMN IF NOT EXISTS`, making the entire migration idempotent.
- **Mixed PK/FK types**: `thought_edges.id` is `BIGSERIAL` (integer surrogate), but `from_thought_id`/`to_thought_id` are `UUID` to match `public.thoughts.id`. This is intentional and documented in the SQL header.

## Non-Obvious Details

- **RLS mirrors `public.thoughts`**: `thought_edges` is service-role-only because rows expose derived relationships between private thoughts. If `authenticated` read were allowed here while `thoughts` restricts it, clients could infer private thought relationships indirectly. Any future relaxation must be an explicit product decision.
- **Upsert NULL-wins rule**: When two classifiers report conflicting `valid_until` values and one is NULL, NULL wins (meaning "still current"). This is the opposite of a typical GREATEST() call and is intentional — a concrete expiry date should not override an open-ended "still current" classification.
- **`valid_from` smallest-wins rule**: On conflict, the earlier `valid_from` is kept (or the non-NULL value if one side is NULL), so the earliest known start of a relationship is preserved across re-classification runs.
- **Prerequisite enforcement at migration time**: The schema opens with a `DO $$ ... RAISE EXCEPTION` block that fails fast if `public.thoughts` or `public.edges` do not exist, surfacing a clear message rather than a cryptic FK error.
- **PostgREST cache reload**: `NOTIFY pgrst, 'reload schema'` is included at the end of the migration so the REST API picks up the new table and columns without a manual server restart.

## Related Modules

- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution, ephemeral IDs)
- **[extensions/family-calendar](../../extensions/family-calendar/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (NULL family_member_id for household-wide events, Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution)
- **[extensions/home-maintenance](../../extensions/home-maintenance/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution, frequency_days=NULL for one-time tasks)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), Symmetric relation canonical ordering, support_count evidence accumulation)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), support_count, support_count evidence accumulation, symmetric relation canonicalization)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), co_occurs_with exclusion, support_count evidence accumulation)
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Orphan row, Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Anchor date/time, Dynamic loop rescheduling, Temporal validity with NULL semantics, decay_weight)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Silent-on-miss contract, Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution)
- **[recipes/ob-graph](../../recipes/ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, support_count evidence accumulation, traverse_graph (outgoing recursive CTE))
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Provides public.thought_edges, public.thought_edges_upsert (RPC), public.edges.valid_from (column), ... consumed by this module
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, Temporal validity with NULL semantics, decay_weight)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), derived_from edges, support_count evidence accumulation)
- **[schemas](../CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), Two-tier edge model (entity edges vs thought edges), support_count evidence accumulation)
- **[schemas/enhanced-thoughts](../enhanced-thoughts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), metadata-overlap thought connections, support_count evidence accumulation)
- **[schemas/entity-extraction](../entity-extraction/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), support_count evidence accumulation)
