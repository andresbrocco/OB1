# grok-export-import

> Import xAI Grok conversation exports into Open Brain as searchable thoughts with embeddings.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Supabase project URL | — | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key for upsert access | — | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key used to generate embeddings | — | Yes |
| `EMBEDDING_MODEL` | Embedding model routed through OpenRouter | `openai/text-embedding-3-small` | No |

Copy `.env.example` to `.env` and fill in values before running.

### Commands

```bash
# Preview what would be imported without writing to the database
npm run dry-run -- /path/to/grok-export.json

# Run a full import
npm run import -- /path/to/grok-export.json

# Import with pagination flags (direct node invocation)
node import-grok.mjs /path/to/grok-export.json --skip 50 --limit 100

# Combine dry-run with pagination to inspect a subset
node import-grok.mjs /path/to/grok-export.json --dry-run --skip 0 --limit 10
```

### CLI Flags

| Flag | Description |
|------|-------------|
| `--dry-run` | Print what would be imported; skip all DB and embedding calls |
| `--skip N` | Skip the first N conversations in the export file |
| `--limit N` | Process at most N conversations |

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables |
| `package.json` | NPM scripts (`import`, `dry-run`) and dependency declarations |

### Database Tables

| Table | Purpose |
|-------|---------|
| `thoughts` | Destination for imported conversations via the `upsert_thought` RPC |

Conversations are inserted using the `upsert_thought` Supabase RPC with `source_type: "grok_import"`. Content fingerprinting (SHA-256) prevents duplicate imports on re-runs.

### Prerequisites

- Node.js >= 18 (native `fetch` required)
- A Grok export JSON file (exported from [x.ai](https://x.ai) account settings)
- An Open Brain Supabase project with the `upsert_thought` RPC deployed
- An OpenRouter account with access to an embedding model

## Common Tasks

### First-time setup

```bash
cd recipes/grok-export-import
cp .env.example .env
# Edit .env with your Supabase and OpenRouter credentials
npm install
```

### Preview an export before importing

```bash
npm run dry-run -- /path/to/grok-export.json
```

Dry-run lists each conversation title and character count without calling OpenRouter or writing to the database.

### Import a full export

```bash
npm run import -- /path/to/grok-export.json
```

Progress is logged per conversation in the format:

```
[12/200] inserted: #4821 "How does RAG work?"
[13/200] updated:  #318  "Protein folding explained"
```

### Import a large export in batches

```bash
# First batch: conversations 1-100
node import-grok.mjs /path/to/grok-export.json --skip 0 --limit 100

# Second batch: conversations 101-200
node import-grok.mjs /path/to/grok-export.json --skip 100 --limit 100
```

### Override the embedding model

Set `EMBEDDING_MODEL` in `.env` before running:

```bash
EMBEDDING_MODEL=openai/text-embedding-3-large npm run import -- /path/to/grok-export.json
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Missing required env vars` on startup | `.env` not present or incomplete | Copy `.env.example` to `.env` and fill in all three required values |
| `Usage: node import-grok.mjs ...` error | No file path argument passed | Pass the export file path as the first positional argument |
| `Embedding failed: 401` | Invalid or missing `OPENROUTER_API_KEY` | Verify the key at [openrouter.ai/keys](https://openrouter.ai/keys) |
| `upsert_thought failed` | RPC not deployed or wrong Supabase credentials | Confirm the `upsert_thought` function exists in your Supabase project and that `SUPABASE_SERVICE_ROLE_KEY` has the correct permissions |
| Conversations show as `skipped` in output | Parsed content under 100 characters | These are empty or near-empty conversations; safe to ignore |
| Truncation notice `[... truncated]` in DB | Conversation text exceeded 30,000 characters | Expected behavior; the stored thought covers the first 30 KB of the transcript |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this recipe
- [../recipes/](../) — Other Open Brain recipes
