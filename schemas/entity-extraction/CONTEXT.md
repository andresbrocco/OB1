# CONTEXT.md — Entity Extraction Schema

## Purpose

Extends the Open Brain database with a knowledge graph layer: canonical entities, typed relationships between them, evidence links back to thoughts, an async processing queue, and an audit log for merge/consolidation operations. Entity population is driven by an automatic trigger on the `thoughts` table and fulfilled by an external worker.

## Responsibility Boundaries

- **Owns**: The `entities`, `edges`, `thought_entities`, `entity_extraction_queue`, and `consolidation_log` tables; the `queue_entity_extraction` trigger function; RLS policies and grants for all five tables.
- **Delegates to**: The entity extraction worker (external — see `integrations/entity-extraction-worker`) to actually read queue rows, call an LLM, and write results back into `entities`, `edges`, and `thought_entities`.
- **Does not handle**: Entity extraction logic, LLM calls, deduplication merges (those are worker responsibilities and are only audited here via `consolidation_log`).

## Key Concepts

- **Canonical entity / normalized name**: Each entity has both a `canonical_name` (display form) and a `normalized_name` (lowercase, trimmed). Deduplication uniqueness is enforced on `(entity_type, normalized_name)`, not on `canonical_name`. Aliases capture alternate surface forms.
- **Edge support count**: The `edges.support_count` column accumulates the number of thought co-occurrences that evidence a relationship. This is an incrementing signal, not a static flag.
- **Thought-entity link with mention role**: `thought_entities` is not a plain junction table — it carries `mention_role` (e.g., "mentioned", "subject"), `confidence`, and `evidence` JSONB, making it an evidence record rather than a bare association.
- **Async queue semantics**: `entity_extraction_queue` is a one-row-per-thought queue with status lifecycle `pending → processing → complete / failed / skipped`. The trigger resets an existing row to `pending` only when `source_fingerprint` has actually changed (content-addressed re-queue), preventing redundant processing on unrelated `UPDATE`s.
- **Consolidation log**: Tracks dedup merges and bio synthesis operations by recording `survivor_id` / `loser_id` pairs and a `details` JSONB blob. It is an append-only audit trail, not a live operational table.

## Non-Obvious Details

- **Hard prerequisite guard**: The migration opens with a `DO $$ … RAISE EXCEPTION` block that aborts immediately if `thoughts.content_fingerprint` does not exist. This prevents a silent install that would then crash on the first `INSERT INTO thoughts`. You must apply docs/01-getting-started.md Step 2.6 first.
- **Trigger skips generated artifacts**: The `queue_entity_extraction` function checks `NEW.metadata->>'generated_by'` and silently skips queuing if the row is a system-generated artifact (consolidation output, bio synthesis, etc.), preventing extraction feedback loops.
- **RLS scaffolded for future multi-tenancy**: Authenticated users receive SELECT-only access with `USING (true)` — no `user_id` filter yet. The policy comments explicitly note where to substitute `auth.uid() = user_id` when multi-tenancy is wired. `anon` has zero access.
- **Backfill is opt-in and commented out**: The trigger only fires on INSERT/UPDATE, so existing thoughts are not automatically queued. A commented-out `INSERT … ON CONFLICT DO NOTHING` block at the end of the file must be run manually once against an existing brain.
- **Idempotent by design**: All DDL uses `IF NOT EXISTS` / `OR REPLACE` / `DROP … IF EXISTS` / `ON CONFLICT DO UPDATE`, making the script safe to re-run.

## Related Modules

- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Async queue with content-addressed re-queue)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Post-search filter extraction, Thought-entity mention role and evidence)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, ExtractionCostCapError, Thought-entity mention role and evidence, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, _enrichment_status)
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Two-phase pipeline)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, Two-Prompt Extraction Sequence)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Async queue with content-addressed re-queue, Retrospective Mode)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, co_occurs_with exclusion)
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Async queue with content-addressed re-queue, Cursor-based resumability)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Two-phase pipeline)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Self-improvement protocol)
- **[recipes/ob-graph](../../recipes/ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Pending person confirmation, Thought-entity mention role and evidence, Three-pass person resolution)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Enrichment versioning, Thought-entity mention role and evidence)
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Provides public.entities, public.edges, public.thought_entities, ... consumed by this module
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Parallel capture pipeline)
- **[recipes/wiki-compiler](../../recipes/wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Phase toggles)
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Async queue with content-addressed re-queue, Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, derived_from edges)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Async queue with content-addressed re-queue, Checkpoint + Entry Separation)
- **[schemas](../CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Entity extraction queue with auto-trigger, Thought-entity mention role and evidence)
- **[schemas/enhanced-thoughts](../enhanced-thoughts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, metadata-overlap thought connections)
- **[schemas/typed-reasoning-edges](../typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), support_count evidence accumulation)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Async queue with content-addressed re-queue, Retrospective mode)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Async queue with content-addressed re-queue, Harness, Harness primitives)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, Thread extraction)
