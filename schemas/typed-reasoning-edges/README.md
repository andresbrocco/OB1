# typed-reasoning-edges

> SQL schema extension that adds a typed reasoning edge graph and temporal validity to the Open Brain knowledge graph.

## Quick Reference

### Database Tables

| Table | Purpose |
|-------|---------|
| `public.thought_edges` | Typed, directional semantic relationships between thoughts with temporal validity and confidence scoring |

### Schema Changes

| Target | Change |
|--------|--------|
| `public.thought_edges` | New table — created by this schema |
| `public.edges` | Altered — adds `valid_from`, `valid_until`, `decay_weight` columns |

### RPC Functions

| Function | Exposed Via | Description |
|----------|-------------|-------------|
| `public.thought_edges_upsert(...)` | `POST /rpc/thought_edges_upsert` | Insert or (on conflict) bump `support_count` and refresh temporal bounds |

### Relation Types

| Relation | Meaning |
|----------|---------|
| `supports` | A strengthens or provides evidence for B |
| `contradicts` | A disagrees with or disproves B |
| `evolved_into` | A was replaced by a refined/updated B |
| `supersedes` | A is the newer replacement for B (decisions/versions) |
| `depends_on` | A is conditional on B being true |
| `related_to` | Generic fallback when no specific label fits |

### Prerequisites

- `public.thoughts` table (from `docs/01-getting-started.md`)
- `schemas/entity-extraction/schema.sql` applied (provides `public.edges` table targeted by temporal-validity columns)
- Supabase project with PostgREST enabled

## Applying the Schema

```bash
# Apply via Supabase SQL editor or psql
psql "$DATABASE_URL" -f schemas/typed-reasoning-edges/schema.sql

# Or paste the contents of schema.sql directly into the Supabase SQL editor
```

The migration is fully idempotent — safe to re-apply. If prerequisites are missing, it fails fast with a descriptive error before making any changes.

## Common Tasks

### Insert a reasoning edge manually

```sql
INSERT INTO public.thought_edges (
  from_thought_id,
  to_thought_id,
  relation,
  confidence,
  classifier_version
)
VALUES (
  '<uuid-of-thought-a>',
  '<uuid-of-thought-b>',
  'supports',
  0.87,
  'manual-v1'
);
```

### Upsert via PostgREST (accumulate evidence)

```bash
curl -X POST "https://<project>.supabase.co/rest/v1/rpc/thought_edges_upsert" \
  -H "Authorization: Bearer <service_role_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "p_from_thought_id": "<uuid-a>",
    "p_to_thought_id": "<uuid-b>",
    "p_relation": "contradicts",
    "p_confidence": 0.75,
    "p_support_count": 1,
    "p_classifier_version": "v1.0",
    "p_valid_from": null,
    "p_valid_until": null,
    "p_metadata": {}
  }'
```

### Query all current outgoing edges from a thought

```sql
SELECT te.*, t.content AS to_thought_content
FROM public.thought_edges te
JOIN public.thoughts t ON te.to_thought_id = t.id
WHERE te.from_thought_id = '<uuid>'
  AND te.valid_until IS NULL
ORDER BY te.confidence DESC;
```

### Query edges by relation type with decay filter

```sql
SELECT *
FROM public.thought_edges
WHERE relation = 'supports'
  AND (decay_weight IS NULL OR decay_weight > 0.5)
  AND valid_until IS NULL;
```

### Mark an edge as expired

```sql
UPDATE public.thought_edges
SET valid_until = now()
WHERE from_thought_id = '<uuid-a>'
  AND to_thought_id = '<uuid-b>'
  AND relation = 'supersedes';
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `RAISE EXCEPTION: requires the public.thoughts table` | Core thoughts table not yet created | Run `docs/01-getting-started.md` setup first |
| `RAISE EXCEPTION: requires the public.edges table` | `schemas/entity-extraction/schema.sql` not applied | Apply entity-extraction schema, then re-run this migration |
| `duplicate key value violates unique constraint` on direct INSERT | Same `(from_thought_id, to_thought_id, relation)` triple already exists | Use `thought_edges_upsert` RPC instead of a plain INSERT |
| `permission denied for table thought_edges` | Caller is not `service_role` | Use `service_role` key; `authenticated` and `anon` are explicitly revoked |
| PostgREST does not reflect new table/columns | Schema cache stale | Schema emits `NOTIFY pgrst, 'reload schema'` at commit; if still stale, restart PostgREST manually |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [schemas/entity-extraction](../entity-extraction) — Provides the `public.edges` table extended by this schema
- [recipes/typed-edge-classifier](../../recipes/typed-edge-classifier) — Worker that populates `thought_edges` automatically
- [schemas/README.md](../README.md) — Schemas overview
