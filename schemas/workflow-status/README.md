# Workflow Status

> Adds `status` and `status_updated_at` columns to the `thoughts` table, enabling kanban-style workflow state tracking for tasks and ideas.

## Quick Reference

### Database Tables

| Table | Columns Added | Purpose |
|-------|--------------|---------|
| `thoughts` | `status TEXT` | Workflow state (e.g., `new`, `inbox`, `processed`, `archived`) |
| `thoughts` | `status_updated_at TIMESTAMPTZ` | Timestamp of last status change |

**Index created:** `idx_thoughts_status` — partial index on `thoughts(status) WHERE status IS NOT NULL` for fast status filtering.

### Configuration

| File | Purpose |
|------|---------|
| `migration.sql` | Adds columns, index, and backfills existing task/idea rows |

### Prerequisites

- Supabase project with Open Brain core set up (the `thoughts` table must already exist)
- Supabase SQL editor or `psql` access

## Common Tasks

### Apply the Migration

Run once against your Supabase project. The migration is idempotent — safe to run multiple times.

**Via Supabase SQL Editor:**

1. Open your project at [supabase.com](https://supabase.com)
2. Navigate to **SQL Editor**
3. Paste the contents of `migration.sql` and click **Run**

**Via psql:**

```bash
psql "$DATABASE_URL" -f migration.sql
```

### Query Thoughts by Status

```sql
-- All open tasks
SELECT id, content, status, status_updated_at
FROM thoughts
WHERE status = 'new'
  AND metadata->>'type' = 'task'
ORDER BY status_updated_at DESC;

-- All statuses in use
SELECT status, COUNT(*) AS count
FROM thoughts
WHERE status IS NOT NULL
GROUP BY status
ORDER BY count DESC;
```

### Update a Thought's Status

```sql
UPDATE thoughts
SET status = 'processed',
    status_updated_at = now()
WHERE id = '<thought-id>';
```

### Reset Status (for re-processing)

```sql
UPDATE thoughts
SET status = 'new',
    status_updated_at = now()
WHERE metadata->>'type' IN ('task', 'idea')
  AND status = 'processed';
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `column "status" already exists` | Migration run previously | Safe to ignore — migration uses `IF NOT EXISTS` |
| Backfill query updates 0 rows | No `task` or `idea` type thoughts exist yet | Expected behavior — status will be set when relevant thoughts are added |
| Slow queries filtering by status | Index not created | Confirm `idx_thoughts_status` exists: `\d thoughts` in psql or check Supabase indexes panel |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [../README.md](../README.md) — Schemas category overview
