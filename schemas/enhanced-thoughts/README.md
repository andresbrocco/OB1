# Enhanced Thoughts

> SQL schema extension that adds type classification, importance scoring, source tracking, and advanced query RPCs to the core `thoughts` table.

## Quick Reference

### Commands

```bash
# Apply the schema (run once against your Supabase project)
psql "$DATABASE_URL" -f schema.sql

# Or via Supabase CLI
supabase db push --db-url "$DATABASE_URL" < schema.sql
```

The script is fully idempotent — safe to run multiple times. All `ALTER TABLE` statements use `IF NOT EXISTS` and all functions use `CREATE OR REPLACE`.

### Database Tables

| Table | Change | Purpose |
|-------|--------|---------|
| `thoughts` | `type TEXT` | Classifies thought (idea, task, person_note, reference, decision, lesson, meeting, journal) |
| `thoughts` | `importance SMALLINT DEFAULT 3` | 1–10 importance score used in search ranking |
| `thoughts` | `quality_score NUMERIC(5,2) DEFAULT 50` | 0–100 quality score used in search ranking |
| `thoughts` | `sensitivity_tier TEXT DEFAULT 'standard'` | Access control tier (standard, restricted) |
| `thoughts` | `source_type TEXT` | Originating source (e.g., slack, email, manual) |
| `thoughts` | `enriched BOOLEAN DEFAULT false` | Whether enrichment pipeline has processed this row |

### Indexes Added

| Index | Column(s) | Type | Purpose |
|-------|-----------|------|---------|
| `idx_thoughts_type` | `type` | B-tree | Filter by thought type |
| `idx_thoughts_importance` | `importance DESC` | B-tree | Sort by importance |
| `idx_thoughts_source_type` | `source_type` | B-tree | Filter by source |
| `idx_thoughts_content_tsvector` | `content` (tsvector) | GIN | Full-text search acceleration |

### RPC Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `search_thoughts_text(p_query, p_limit, p_filter, p_offset)` | `TABLE` | Full-text search with boolean operators, ILIKE fallback, pagination, and relevance ranking |
| `brain_stats_aggregate(p_since_days, p_exclude_restricted)` | `JSONB` | Aggregate stats: total count, top types, top topics |
| `get_thought_connections(p_thought_id, p_limit, p_exclude_restricted)` | `TABLE` | Finds related thoughts by shared metadata topics and people |

All three RPCs are granted to `authenticated`, `anon`, and `service_role`.

### Prerequisites

- Supabase project with the core Open Brain `thoughts` table already created
- PostgreSQL 14+ (for `websearch_to_tsquery` and `jsonb_array_elements_text`)
- pgvector extension (provided by Supabase by default)

## Common Tasks

### Apply the Schema

```bash
# Using psql directly
psql "postgres://postgres:<password>@<host>:5432/postgres" -f schema.sql

# Using Supabase CLI
supabase db reset  # dev only — applies all migrations
```

### Search Thoughts via RPC

```sql
-- Basic full-text search
SELECT * FROM search_thoughts_text('project planning', 25, '{}', 0);

-- Search with metadata filter (type = 'idea')
SELECT * FROM search_thoughts_text('startup', 10, '{"type": "idea"}', 0);

-- Boolean operators: quoted phrase, AND, OR, NOT
SELECT * FROM search_thoughts_text('"machine learning" OR "deep learning"', 20, '{}', 0);

-- Paginated (page 2, 25 per page)
SELECT * FROM search_thoughts_text('leadership', 25, '{}', 25);
```

```bash
# Via Supabase REST API
curl -X POST "https://<project>.supabase.co/rest/v1/rpc/search_thoughts_text" \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"p_query": "project planning", "p_limit": 25, "p_filter": {}, "p_offset": 0}'
```

### Get Brain Statistics

```sql
-- Last 30 days, excluding restricted
SELECT brain_stats_aggregate(30, true);

-- All-time stats including restricted
SELECT brain_stats_aggregate(0, false);
```

```bash
curl -X POST "https://<project>.supabase.co/rest/v1/rpc/brain_stats_aggregate" \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"p_since_days": 30, "p_exclude_restricted": true}'
```

Response shape:
```json
{
  "total": 1234,
  "top_types": [{"type": "idea", "count": 400}, ...],
  "top_topics": [{"topic": "AI", "count": 120}, ...]
}
```

### Find Connected Thoughts

```sql
-- Get up to 20 thoughts related to a given thought ID
SELECT * FROM get_thought_connections(
  'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
  20,
  true
);
```

```bash
curl -X POST "https://<project>.supabase.co/rest/v1/rpc/get_thought_connections" \
  -H "apikey: <anon-key>" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"p_thought_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", "p_limit": 20, "p_exclude_restricted": true}'
```

### Backfill Existing Rows

The schema includes a safe backfill block that runs automatically as part of `schema.sql`. It populates `type` from `metadata->>'type'` and `source_type` from `metadata->>'source'` for existing rows where those columns are `NULL`. The `WHERE ... IS NULL` guard makes it safe to re-run.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `column "type" of relation "thoughts" already exists` | Schema was partially applied before `IF NOT EXISTS` guards existed | Script is now idempotent — this error should not occur with current version |
| `search_thoughts_text` returns no rows despite matching content | `websearch_to_tsquery` tokenized query differently than expected | Try simpler terms; the ILIKE fallback activates automatically when GIN results are insufficient |
| `get_thought_connections` returns empty set | Source thought has no `topics` or `people` in its `metadata` JSONB | Enrich the thought metadata first or check `metadata` column directly |
| `brain_stats_aggregate` returns `total: 0` | `p_exclude_restricted: true` and all rows have `sensitivity_tier = 'restricted'` | Pass `p_exclude_restricted: false` or update tier values |
| PostgREST does not see new columns after apply | Schema cache not refreshed | Script issues `NOTIFY pgrst, 'reload schema'` at the end; if still stale, restart PostgREST |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [../README.md](../README.md) — Schemas category overview
- [../../primitives/rls/README.md](../../primitives/rls/README.md) — Row-level security patterns for `sensitivity_tier`
