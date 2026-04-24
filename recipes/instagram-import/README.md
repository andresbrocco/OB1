# Instagram Import

> Import Instagram data exports — DM conversations, comments, and post captions — into Open Brain as searchable thoughts.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Your Supabase project URL | — | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (bypass RLS) | — | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key for embedding generation | — | Yes |
| `EMBEDDING_MODEL` | Embedding model override via OpenRouter | `openai/text-embedding-3-small` | No |

Copy `.env.example` to `.env` and fill in values before running.

### Commands

```bash
# Preview what would be imported without writing to the database
npm run dry-run -- /path/to/instagram-export

# Run a full import
npm run import -- /path/to/instagram-export

# Import only specific content types
node import-instagram.mjs /path/to/instagram-export --types messages,comments

# Skip the first N items and limit total items processed
node import-instagram.mjs /path/to/instagram-export --skip 50 --limit 100

# Combine flags
node import-instagram.mjs /path/to/instagram-export --types posts --dry-run
```

#### CLI Flags

| Flag | Description |
|------|-------------|
| `--dry-run` | Preview import without writing any records |
| `--types <list>` | Comma-separated content types: `messages`, `comments`, `posts` (default: all three) |
| `--skip N` | Skip the first N items |
| `--limit N` | Process at most N items |

### Configuration

| File | Purpose |
|------|---------|
| `.env` | Runtime credentials and model override (created from `.env.example`) |
| `.env.example` | Template showing all supported variables |
| `package.json` | Node.js dependencies and `import`/`dry-run` script shortcuts |

### Database Tables

| Table | Purpose |
|-------|---------|
| `thoughts` | Destination for all imported records via `upsert_thought` RPC |

Records are inserted with `source_type: "instagram_import"`, `sensitivity_tier: "personal"`, and a SHA-256 content fingerprint to prevent duplicate imports.

### Prerequisites

- Node.js 18+
- An Instagram data export folder (request from Instagram: Settings → Your activity → Download your information). The script expects the `your_instagram_activity/` directory to exist somewhere within the export folder (searched up to 3 levels deep).
- A running Open Brain Supabase instance with the `upsert_thought` RPC available.
- An OpenRouter API key with access to an embedding model.

## Common Tasks

### Request Your Instagram Export

1. Open Instagram → Settings → Your activity → Download your information.
2. Select **JSON format** (not HTML).
3. Request the download and wait for the email link.
4. Unzip the archive to a local folder (e.g., `~/Downloads/instagram-export`).

### Run a Dry Run First

```bash
cp .env.example .env
# Edit .env with your credentials

npm run dry-run -- ~/Downloads/instagram-export
```

The dry run prints what would be imported without touching the database, so you can verify item counts and titles before committing.

### Full Import

```bash
npm run import -- ~/Downloads/instagram-export
```

Output per record:

```
[12/47] inserted: #1045 "Instagram DM: alice (82 messages)"
[13/47] updated:  #1046 "Instagram comments (batch 1)"
```

### Import Only One Content Type

```bash
# Only DM conversations
node import-instagram.mjs ~/Downloads/instagram-export --types messages

# Only post captions
node import-instagram.mjs ~/Downloads/instagram-export --types posts
```

### Resume a Partial Import

Use `--skip` to pick up where a previous run left off:

```bash
node import-instagram.mjs ~/Downloads/instagram-export --skip 200
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Could not find 'your_instagram_activity' directory` | Export path is wrong or export is in HTML format | Re-request export in JSON format; point the script at the unzipped folder root |
| `Embedding failed: 401` | `OPENROUTER_API_KEY` is missing or invalid | Check `.env` and verify the key at openrouter.ai |
| `upsert_thought failed: ...` | Supabase credentials wrong or `upsert_thought` RPC not installed | Verify `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`; confirm Open Brain schema is applied |
| Garbled characters in names/text | Meta encodes text as latin1-interpreted UTF-8 | The script applies `fixMetaEncoding` automatically; no action needed |
| Import exits with 0 items | Export has no messages with 3+ messages, no comments over 10 chars, no post captions over 10 chars | Verify the export contains actual activity data |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this recipe
- [../recipes/](../) — Other available recipes

