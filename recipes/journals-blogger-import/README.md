# journals-blogger-import

> Import Google Blogger Atom XML exports into Open Brain as journal thoughts with embeddings.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Supabase project URL | — | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (bypasses RLS) | — | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key for embedding generation | — | Yes |
| `EMBEDDING_MODEL` | Embedding model passed to OpenRouter | `openai/text-embedding-3-small` | No |

Copy `.env.example` to `.env` and fill in your values before running.

### Commands

```bash
# Preview what would be imported without writing to the database
npm run dry-run -- /path/to/blogger-exports

# Run a full live import
npm run import -- /path/to/blogger-exports

# Import with pagination controls
node import-blogger.mjs /path/to/blogger-exports --skip 100 --limit 50

# Dry run with pagination
node import-blogger.mjs /path/to/blogger-exports --dry-run --skip 0 --limit 20
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables |
| `package.json` | Node.js dependencies and npm script shortcuts |

### Database Tables

| Table | Purpose |
|-------|---------|
| `thoughts` | Destination for imported blog posts and comments (via `upsert_thought` RPC) |

### Prerequisites

- Node.js (ESM support required — v18+)
- Supabase project with the Open Brain `thoughts` table and `upsert_thought` RPC
- OpenRouter account with access to an embedding model
- Google Blogger export in Atom (`.atom`) format — download from Blogger Settings → Manage Blog → Back up content

## Common Tasks

### Export your Blogger data

1. Log in to [Blogger](https://www.blogger.com)
2. Go to **Settings** → **Other** → **Back up content**
3. Download the Atom XML file(s) — one per blog
4. Place all `.atom` files in a directory (e.g., `~/blogger-exports/`)

### Preview before importing

```bash
cp .env.example .env
# Fill in .env with your credentials

npm install
npm run dry-run -- ~/blogger-exports
```

The dry-run prints each entry that would be imported without touching the database.

### Run a full import

```bash
npm run import -- ~/blogger-exports
```

Progress is printed per entry in the format:

```
[42/350] inserted: #1234 "My Blog Post Title"
[43/350] duplicate: #1200 "An Older Post"
```

### Import a subset (large archives)

Use `--skip` and `--limit` to process the archive in batches:

```bash
# First batch
node import-blogger.mjs ~/blogger-exports --skip 0 --limit 200

# Second batch
node import-blogger.mjs ~/blogger-exports --skip 200 --limit 200
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Missing required env vars` on startup | `.env` file not present or incomplete | Copy `.env.example` to `.env` and fill in all three required vars |
| `Embedding failed: 401` | Invalid or missing `OPENROUTER_API_KEY` | Verify the key at [openrouter.ai/keys](https://openrouter.ai/keys) |
| `upsert_thought failed` | `upsert_thought` RPC not installed or wrong Supabase URL/key | Confirm the Open Brain core schema is deployed to your project |
| No `.atom` files found | Wrong directory path or files use a different extension | Blogger exports always produce `.atom` files — check the path and file names |
| Entries skipped silently | Content under 20 characters, or entry is not a `post` or `comment` kind | Expected — templates, settings entries, and near-empty posts are filtered out |
| Import stalls mid-run | OpenRouter rate limit or transient network error | Re-run with `--skip N` to resume from the failed offset |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [../recipes/](../) — Other available recipes

