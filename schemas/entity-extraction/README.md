# schemas/entity-extraction

> SQL schema extension that adds five knowledge graph tables for named entity extraction, relationship tracking, async processing, and audit logging.

## Quick Reference

### Database Tables

| Table | Purpose |
|-------|---------|
| `entities` | Canonical graph nodes — people, projects, topics, tools, organizations, places |
| `edges` | Typed relationships between entities (co_occurs_with, works_on, uses, related_to, member_of, located_in) |
| `thought_entities` | Junction linking thoughts to entities, with mention role and extraction confidence |
| `entity_extraction_queue` | Async queue for thoughts awaiting entity extraction processing |
| `consolidation_log` | Audit trail for dedup merges, metadata fixes, and bio synthesis operations |

### Key Columns

**`entities`**

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | Primary key |
| `entity_type` | TEXT | `person`, `project`, `topic`, `tool`, `organization`, `place` |
| `canonical_name` | TEXT | Display name |
| `normalized_name` | TEXT | Lowercase, trimmed — used for dedup (unique with `entity_type`) |
| `aliases` | JSONB | Alternative names for this entity |
| `metadata` | JSONB | Extensible metadata bag |
| `first_seen_at` / `last_seen_at` | TIMESTAMPTZ | Temporal tracking |

**`edges`**

| Column | Type | Notes |
|--------|------|-------|
| `from_entity_id` / `to_entity_id` | BIGINT | FK to `entities` — cascade deletes |
| `relation` | TEXT | `co_occurs_with`, `works_on`, `uses`, `related_to`, `member_of`, `located_in` |
| `support_count` | INT | How many thoughts support this edge |
| `confidence` | NUMERIC(3,2) | 0.00–1.00 |

**`entity_extraction_queue`**

| Column | Type | Notes |
|--------|------|-------|
| `thought_id` | UUID | PK + FK to `thoughts` |
| `status` | TEXT | `pending`, `processing`, `complete`, `failed`, `skipped` |
| `attempt_count` | INT | Retry tracking |
| `last_error` | TEXT | Last worker error message |
| `source_fingerprint` | TEXT | `content_fingerprint` snapshot at queue time — prevents no-op reprocessing |
| `worker_version` | TEXT | Version of worker that processed this row |

### Trigger

`trg_queue_entity_extraction` fires `AFTER INSERT OR UPDATE OF content, metadata` on `public.thoughts`. It inserts or resets a `pending` row in `entity_extraction_queue`, skipping system-generated thoughts (`metadata->>'generated_by'` is non-null) and no-op fingerprint changes.

### Prerequisites

- Supabase project with `public.thoughts` table from the base OB1 setup
- `content_fingerprint` column on `thoughts` — from `docs/01-getting-started.md` Step 2.6 (the migration hard-fails with a clear error if missing)
- `integrations/entity-extraction-worker` to consume the queue

### Schema File

| File | Purpose |
|------|---------|
| `schema.sql` | Single idempotent migration — safe to run multiple times (`CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, `CREATE OR REPLACE FUNCTION`) |

## Common Tasks

### Apply the schema to a new brain

```sql
-- Run in Supabase SQL Editor (or psql)
\i schema.sql
```

### Backfill existing thoughts into the queue

The trigger only fires on new inserts or updates. To queue pre-existing thoughts, uncomment the backfill block at the bottom of `schema.sql`:

```sql
INSERT INTO public.entity_extraction_queue
  (thought_id, status, source_fingerprint, source_updated_at)
SELECT id, 'pending', content_fingerprint, updated_at
FROM public.thoughts
WHERE (metadata->>'generated_by') IS NULL
ON CONFLICT (thought_id) DO NOTHING;
```

### Check extraction queue status

```sql
SELECT status, COUNT(*) AS count
FROM public.entity_extraction_queue
GROUP BY status
ORDER BY count DESC;
```

### View entities for a specific thought

```sql
SELECT e.entity_type, e.canonical_name, te.mention_role, te.confidence
FROM public.thought_entities te
JOIN public.entities e ON e.id = te.entity_id
WHERE te.thought_id = '<your-thought-uuid>';
```

### Find relationships for an entity

```sql
SELECT
  e1.canonical_name AS from_entity,
  ed.relation,
  e2.canonical_name AS to_entity,
  ed.support_count,
  ed.confidence
FROM public.edges ed
JOIN public.entities e1 ON e1.id = ed.from_entity_id
JOIN public.entities e2 ON e2.id = ed.to_entity_id
WHERE e1.canonical_name ILIKE '%<name>%'
   OR e2.canonical_name ILIKE '%<name>%';
```

### Check consolidation audit history

```sql
SELECT operation, survivor_id, loser_id, details, created_at
FROM public.consolidation_log
ORDER BY created_at DESC
LIMIT 50;
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| Migration fails with `entity-extraction requires the content_fingerprint column` | `thoughts.content_fingerprint` does not exist | Run `docs/01-getting-started.md` Step 2.6 first, then re-apply `schema.sql` |
| Existing thoughts never appear in the queue | Trigger only fires on INSERT/UPDATE | Run the backfill query in the Common Tasks section above |
| Queue row stays `pending` with no progress | `integrations/entity-extraction-worker` is not deployed or not running | Deploy and configure the worker — see `integrations/entity-extraction-worker` |
| Queue row status is `failed` | Worker encountered an extraction error | Check `last_error` column: `SELECT thought_id, attempt_count, last_error FROM entity_extraction_queue WHERE status = 'failed'` |
| No-op re-queuing (row not reset after thought update) | `content_fingerprint` did not change | Expected behavior — the trigger skips re-queuing when the fingerprint is identical |
| Dashboard cannot read entity tables | RLS blocks access | Ensure the client uses the `authenticated` role (JWT) or `service_role` key; `anon` has no access by design |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this schema
- [../schemas/README.md](../README.md) — Schemas category overview
- `integrations/entity-extraction-worker` — Worker that consumes `entity_extraction_queue` and populates `entities`, `edges`, `thought_entities`
- `recipes/ob-graph` — Graph visualization recipe built on these tables
- `primitives/rls/` — RLS patterns used in this schema
