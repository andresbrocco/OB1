# X/Twitter Import

> Parse and import a Twitter/X data export archive — tweets, DMs, and Grok chats — into Open Brain as embedded thoughts.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Your Supabase project URL | — | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key for DB writes | — | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key used for embedding generation | — | Yes |
| `EMBEDDING_MODEL` | Embedding model passed to OpenRouter | `openai/text-embedding-3-small` | No |

Copy `.env.example` to `.env` and fill in values before running.

### Commands

```bash
# Preview what would be imported without writing to the database
npm run dry-run -- /path/to/twitter-export

# Run the full import
npm run import -- /path/to/twitter-export

# Import only specific content types
node import-x-twitter.mjs /path/to/twitter-export --types tweets,dms

# Skip the first N items and import up to a limit
node import-x-twitter.mjs /path/to/twitter-export --skip 100 --limit 50
```

### CLI Flags

| Flag | Description |
|------|-------------|
| `--dry-run` | Print what would be imported without writing |
| `--types tweets,dms,grok` | Comma-separated list of content types to import (default: all three) |
| `--skip N` | Skip the first N resolved items |
| `--limit N` | Cap the number of items processed |

### Expected Archive Structure

```
twitter-export/
└── data/
    ├── tweets.js          # or tweet.js
    ├── direct-messages.js # or direct-message.js
    └── grok-conversations.js
```

Files can also be placed in the archive root if no `data/` subdirectory is present.

### Database Tables

| Table | Purpose |
|-------|---------|
| `thoughts` | Destination for all imported records; written via the `upsert_thought` RPC |

### Prerequisites

- Node.js 18+ (ESM support required)
- A Twitter/X data export downloaded from your account settings
- `npm install` run in this directory
- `.env` file populated from `.env.example`

## Common Tasks

### Run a dry run first

Always preview before a live import to confirm item counts and titles look correct.

```bash
cp .env.example .env
# edit .env with real credentials
npm install
npm run dry-run -- /path/to/twitter-export
```

### Import tweets only

```bash
node import-x-twitter.mjs /path/to/twitter-export --types tweets
```

### Import DMs and Grok chats, skip tweets

```bash
node import-x-twitter.mjs /path/to/twitter-export --types dms,grok
```

### Resume a partial import

Use `--skip` to resume from where a previous run left off.

```bash
# If a previous run processed 200 items, resume from item 201
node import-x-twitter.mjs /path/to/twitter-export --skip 200
```

### Override the embedding model

Set `EMBEDDING_MODEL` in `.env` to any model available on OpenRouter.

```bash
EMBEDDING_MODEL=openai/text-embedding-3-large node import-x-twitter.mjs /path/to/twitter-export
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Missing required env vars` on startup | `.env` not loaded or missing keys | Confirm `.env` exists and all three required vars are set |
| `Embedding failed: 401` | Invalid or expired `OPENROUTER_API_KEY` | Regenerate the key in your OpenRouter dashboard |
| `upsert_thought failed` | `upsert_thought` RPC not present in your database | Apply the base Open Brain schema; see `docs/01-getting-started.md` |
| `Total items: 0` after running | Archive path wrong or files not found | Confirm the path contains a `data/` folder with `.js` files |
| Retweets not imported | Intentional — retweets (`RT @`) are filtered out | Only original tweets are stored |
| Tweets shorter than 30 characters skipped | Intentional minimum-length filter | Short tweets are excluded to avoid low-signal noise |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this recipe
- [../recipes/](../) — Other Open Brain recipes
