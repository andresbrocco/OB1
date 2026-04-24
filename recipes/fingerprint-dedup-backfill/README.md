# fingerprint-dedup-backfill

> Three-phase utility to backfill content fingerprints on existing thoughts and remove duplicates: backfill, report, cleanup.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Your Supabase project URL (e.g. `https://your-project-ref.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key — bypasses RLS for bulk operations | Yes |

Set these in `.env` or `.env.local` in this directory, or export them as shell variables. The scripts load `.env` first, then `.env.local`, then fall back to the process environment.

```bash
cp .env.example .env
# Edit .env with your actual values
```

### Commands

```bash
# Install dependencies (none required — pure Node.js ESM with no external packages)
# Node.js >= 18 required (uses native fetch)

# Phase 1 — Backfill fingerprints on all NULL rows
npm run backfill

# Phase 2 — Report duplicates without deleting anything (safe, read-only)
npm run report

# Phase 3 — Delete confirmed duplicate rows and patch remaining orphans
npm run cleanup
```

Direct invocations (bypass npm scripts):

```bash
# Backfill
node backfill-fingerprints.mjs

# Report only (no deletions)
node delete-duplicates.mjs --report-only

# Destructive cleanup
node delete-duplicates.mjs --delete
```

### Configuration

| File | Purpose |
|------|---------|
| `.env` | Primary environment variable file (gitignored) |
| `.env.local` | Local override (gitignored, takes precedence over `.env`) |
| `.env.example` | Template — copy to `.env` and fill in values |
| `backfill-state.json` | Auto-generated cursor state for resumable backfill runs (deleted on completion) |
| `cleanup-state.json` | Auto-generated cursor state for resumable cleanup runs (deleted on completion) |

### Database Tables

| Table | Operation | Description |
|-------|-----------|-------------|
| `thoughts` | PATCH | Writes `content_fingerprint` to rows where it is `NULL` |
| `thoughts` | DELETE | Removes duplicate rows whose fingerprint already exists on a canonical row |

### Prerequisites

- Node.js >= 18 (native `fetch` required — no polyfill)
- A Supabase project with the Open Brain `thoughts` table
- `content_fingerprint` column must exist on `thoughts` (added by the content-fingerprint-dedup primitive)
- Service role key (not anon key) — operations bypass RLS

## Common Tasks

### Run a full deduplication pass (recommended order)

```bash
# Step 1: backfill fingerprints on all unprocessed rows
npm run backfill

# Step 2: preview what would be deleted (no writes)
npm run report

# Step 3: delete confirmed duplicates and patch orphans
npm run cleanup
```

### Resume an interrupted run

Both scripts save a cursor state file (`backfill-state.json` or `cleanup-state.json`) after each batch. If a run is interrupted, simply re-run the same command — it will resume from the last saved cursor automatically.

```bash
# Resume interrupted backfill
npm run backfill

# Resume interrupted cleanup
npm run cleanup
```

### Check how many duplicates exist without deleting

```bash
npm run report
# Output shows "Total rows that would be deleted: N"
# Re-run anytime — it is fully non-destructive
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Missing SUPABASE_URL or SUPABASE_SERVICE_ROLE_KEY` | Credentials not found | Copy `.env.example` to `.env` and fill in both values |
| `Fetch HTTP 401` | Wrong or expired service role key | Re-copy the service role key from Supabase Dashboard → Settings → API |
| `Fetch HTTP 404` | `SUPABASE_URL` points to wrong project or `thoughts` table does not exist | Verify the URL and confirm the `thoughts` table is present |
| `PATCH error: HTTP 409 / 23505` | Fingerprint collision (unique constraint violation) | Expected — the script counts these as `duplicates (skipped)` and continues |
| Script exits immediately with `(no rows) — Done` | All rows already have a fingerprint | Nothing to do; backfill is complete |
| `content_fingerprint` column missing | Primitive not applied | Apply the `content-fingerprint-dedup` schema first |
| Run appears stuck | Large table with slow REST batches | Each batch has a 150–200 ms delay; check progress via cursor output |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this recipe
- [../../primitives/README.md](../../primitives/README.md) — Primitives index
