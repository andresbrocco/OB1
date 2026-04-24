# Google Activity Import

> Import your Google Search, Gmail, Maps, YouTube, and Chrome history from Google Takeout into Open Brain as searchable thoughts.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Your Supabase project URL (e.g., `https://xxxxx.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key from Project Settings → API | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key for LLM summarization and embeddings | Yes (unless `--raw`) |
| `GOOGLE_TAKEOUT_PATH` | Path to your extracted Google Takeout "My Activity" folder | CLI arg |

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

### Commands

```bash
# Preview what would be imported (no database writes)
npm run dry-run
# equivalent: node import-google-activity.mjs ./Takeout/My\ Activity --dry-run --limit 5

# Run full import
npm run import -- ./Takeout/My\ Activity

# Import with date range
node import-google-activity.mjs ./Takeout/My\ Activity --after 2024-01-01 --before 2024-12-31

# Import specific categories only
node import-google-activity.mjs ./Takeout/My\ Activity --categories Search,Gmail

# Skip LLM summarization and insert raw grouped entries
node import-google-activity.mjs ./Takeout/My\ Activity --raw

# Show full thought text during processing
node import-google-activity.mjs ./Takeout/My\ Activity --verbose
```

### CLI Options

| Option | Description | Default |
|--------|-------------|---------|
| `<takeout-path>` | Path to your extracted Google Takeout "My Activity" folder | Required |
| `--dry-run` | Parse, filter, and summarize without writing to the database | Off |
| `--limit N` | Maximum activity-days to process | Unlimited |
| `--after YYYY-MM-DD` | Only process activity after this date | None |
| `--before YYYY-MM-DD` | Only process activity before this date | None |
| `--categories LIST` | Comma-separated category names to process | Search, Gmail, Maps, YouTube, Chrome, Gemini Apps |
| `--raw` | Skip LLM summarization; insert grouped entries as-is | Off |
| `--verbose` | Print full thought text during processing | Off |
| `--help` | Show usage information | — |

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables |
| `google-activity-sync-log.json` | Auto-generated sync log tracking already-processed days (prevents duplicate imports) |

### Database Tables

| Table | Purpose |
|-------|---------|
| `thoughts` | Destination for all imported activity thoughts; rows include `content`, `embedding`, and `metadata` fields |

Metadata written per thought:

| Field | Value |
|-------|-------|
| `source` | `"google_activity"` |
| `google_category` | e.g., `"Search"`, `"Gmail"`, `"Maps"` |
| `google_date` | ISO date string `YYYY-MM-DD` |
| `entry_count` | Number of raw activity entries for that day |

### Prerequisites

- Node.js 18+ (uses native `fetch` and ES modules)
- An active Open Brain setup (Supabase project with the `thoughts` table and `pgvector`)
- An OpenRouter account with an API key ([openrouter.ai/keys](https://openrouter.ai/keys))
- A Google Takeout export with "My Activity" data (JSON format)

## Common Tasks

### Get Your Google Takeout Export

1. Go to [takeout.google.com](https://takeout.google.com)
2. Deselect all, then select only "My Activity"
3. Choose JSON format (not HTML)
4. Download and extract the archive
5. Locate the `Takeout/My Activity/` folder — this is your `<takeout-path>`

### First-Time Dry Run

Run a dry run on a small slice before committing to a full import:

```bash
node import-google-activity.mjs "./Takeout/My Activity" --dry-run --limit 10 --verbose
```

Review the printed thoughts. If the quality looks good, proceed with a live import.

### Full Live Import

```bash
cp .env.example .env
# Fill in SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, OPENROUTER_API_KEY

node import-google-activity.mjs "./Takeout/My Activity"
```

The script writes `google-activity-sync-log.json` after each successfully processed day. Re-running the script is safe — already-processed days are skipped via content hash comparison.

### Import Only Recent Activity

```bash
node import-google-activity.mjs "./Takeout/My Activity" --after 2024-06-01
```

### Re-import Without Summarization (Faster, Lower Cost)

```bash
node import-google-activity.mjs "./Takeout/My Activity" --raw
```

Raw mode skips the GPT-4o-mini summarization step and inserts grouped daily entries directly. Useful for large archives where you want to minimize API cost.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Error: No MyActivity.json files found` | Wrong path or wrong format selected during Takeout export | Point the path to the `My Activity` folder inside the Takeout archive; ensure you exported in JSON (not HTML) format |
| `Error: SUPABASE_URL environment variable required` | Missing env var | Set `SUPABASE_URL` in your `.env` or shell environment |
| `Error: OPENROUTER_API_KEY required for summarization` | Missing API key | Set `OPENROUTER_API_KEY`, or use `--raw` to skip summarization |
| `Warning: Summarization failed (429)` | OpenRouter rate limit | The script retries automatically; reduce throughput with `--limit` or wait and re-run (sync log prevents re-processing) |
| `HTTP 401` from Supabase | Invalid or expired service role key | Regenerate from Supabase Project Settings → API |
| Thoughts are too generic or low signal | Noise filtering is working as intended | Use `--verbose` to review; adjust `--categories` to focus on higher-signal sources |
| Duplicate thoughts after re-run | Sync log was deleted or entries changed | The sync log hashes content per day — deleting it causes full re-import |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this recipe
- [../chatgpt-conversation-import/README.md](../chatgpt-conversation-import/README.md) — Similar pattern for ChatGPT history
- [../email-history-import/README.md](../email-history-import/README.md) — Email history import recipe
