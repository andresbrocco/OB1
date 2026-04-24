# ChatGPT Conversation Import

> Parse a ChatGPT data export, extract 2-5 structured knowledge thoughts per conversation via LLM, and ingest them into Open Brain with enriched metadata and embeddings.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Supabase project URL (e.g. `https://xyz.supabase.co`) | Yes (default mode) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key | Yes (default mode) |
| `OPENROUTER_API_KEY` | OpenRouter API key for LLM extraction and embeddings | Yes |
| `INGEST_URL` | Custom ingest endpoint URL | Only with `--ingest-endpoint` |
| `INGEST_KEY` | Auth key for custom ingest endpoint | Only with `--ingest-endpoint` |
| `USER_ID` | User UUID for multi-tenant RLS | Only with `--store-conversations` |

Copy `.env.example` to `.env` and fill in values, then load with:

```bash
export $(cat .env | xargs)
```

### Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Basic import (Supabase direct insert, all conversations)
python3 import-chatgpt.py path/to/export.zip

# Import from extracted export directory
python3 import-chatgpt.py path/to/extracted-dir/

# Dry run — parse, filter, and extract without ingesting
python3 import-chatgpt.py path/to/export.zip --dry-run

# Filter by date range
python3 import-chatgpt.py path/to/export.zip --after 2024-01-01 --before 2025-01-01

# Limit number of conversations processed
python3 import-chatgpt.py path/to/export.zip --limit 50

# Use custom ingest endpoint instead of Supabase direct insert
python3 import-chatgpt.py path/to/export.zip --ingest-endpoint

# Use local Ollama instead of OpenRouter
python3 import-chatgpt.py path/to/export.zip --model ollama --ollama-model qwen3

# Skip LLM extraction — ingest raw user messages directly
python3 import-chatgpt.py path/to/export.zip --raw

# Also store conversation-level summaries in chatgpt_conversations table
python3 import-chatgpt.py path/to/export.zip --store-conversations

# Write a markdown report of everything imported
python3 import-chatgpt.py path/to/export.zip --report import-report.md

# Focus on a topic category (tech | strategy | personal | creative)
python3 import-chatgpt.py path/to/export.zip --focus tech

# Focus on a free-text topic
python3 import-chatgpt.py path/to/export.zip --focus "machine learning"

# Verbose output — show full extracted thoughts during processing
python3 import-chatgpt.py path/to/export.zip --verbose
```

### All CLI Options

| Option | Description |
|--------|-------------|
| `--dry-run` | Parse, filter, and extract without ingesting |
| `--after YYYY-MM-DD` | Only conversations created after this date |
| `--before YYYY-MM-DD` | Only conversations created before this date |
| `--limit N` | Max conversations to process |
| `--model openrouter\|ollama` | LLM backend (default: `openrouter`) |
| `--ollama-model NAME` | Ollama model name (default: `qwen3`) |
| `--raw` | Skip extraction, ingest user messages directly |
| `--focus PRESET\|TEXT` | Filter to a topic: `tech`, `strategy`, `personal`, `creative`, or free text |
| `--verbose` | Show full thoughts during processing |
| `--report FILE` | Write a markdown import report |
| `--ingest-endpoint` | Use `INGEST_URL`/`INGEST_KEY` instead of Supabase direct |
| `--store-conversations` | Also store conversation-level summaries (requires schema.sql) |
| `--min-messages N` | Override minimum message count filter |
| `--min-words N` | Override minimum word count for borderline filtering |
| `--max-words N` | Skip conversations exceeding N words (default: 50000) |

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for environment variables — copy to `.env` |
| `schema.sql` | Optional schema for `chatgpt_conversations` table (run before `--store-conversations`) |
| `chatgpt-sync-log.json` | Auto-generated sync log tracking previously imported conversations |

### Database Tables

| Table | Purpose | When Used |
|-------|---------|-----------|
| `thoughts` | Core Open Brain table — receives extracted knowledge thoughts | Always |
| `chatgpt_conversations` | Conversation-level summaries with pyramid detail levels | Only with `--store-conversations` |

The `chatgpt_conversations` table must be created by running `schema.sql` in your Supabase SQL Editor before using `--store-conversations`. It includes an HNSW index on `embedding` for semantic search and RLS policies for multi-tenant isolation.

### Prerequisites

- Python 3.10+
- Open Brain Supabase instance (with `thoughts` table and pgvector enabled)
- OpenRouter account and API key (or local Ollama instance)
- ChatGPT data export (`.zip` file or extracted directory from [chatgpt.com/settings](https://chatgpt.com/settings))

## Common Tasks

### First-Time Import

```bash
# 1. Set up environment
cp .env.example .env
# Edit .env with your Supabase URL, service role key, and OpenRouter key
export $(cat .env | xargs)

# 2. Install dependencies
pip install -r requirements.txt

# 3. Dry run to verify parsing
python3 import-chatgpt.py ~/Downloads/chatgpt-export.zip --dry-run --limit 10

# 4. Run the full import
python3 import-chatgpt.py ~/Downloads/chatgpt-export.zip --report import-report.md
```

### Import with Conversation Summaries

Requires running `schema.sql` in Supabase SQL Editor first.

```bash
python3 import-chatgpt.py ~/Downloads/chatgpt-export.zip --store-conversations
```

### Re-import After New Export

The script maintains `chatgpt-sync-log.json` to track previously processed conversations. Re-running will skip already-imported conversations and only process new ones.

```bash
python3 import-chatgpt.py ~/Downloads/chatgpt-export-new.zip
```

### Import Only Work-Related Conversations

```bash
python3 import-chatgpt.py ~/Downloads/chatgpt-export.zip --focus tech
python3 import-chatgpt.py ~/Downloads/chatgpt-export.zip --focus strategy
```

### Use with a Custom Ingest Endpoint

```bash
# Set INGEST_URL and INGEST_KEY in .env, then:
python3 import-chatgpt.py ~/Downloads/chatgpt-export.zip --ingest-endpoint
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `SUPABASE_URL not set` error | Environment variables not loaded | Run `export $(cat .env | xargs)` before the script |
| `chatgpt_conversations` table not found | `schema.sql` not applied | Run `schema.sql` in Supabase SQL Editor before using `--store-conversations` |
| All conversations skipped | Export format unrecognized | Confirm the zip contains `conversations.json` or numbered `conversations-000.json` files |
| Embeddings failing | OpenRouter API key invalid or quota exceeded | Verify `OPENROUTER_API_KEY` is set and has remaining credits |
| Very slow on large export | Large export with many conversations | Use `--limit N` to batch, or `--after`/`--before` to process date ranges incrementally |
| Thoughts too noisy / off-topic | No focus filter set | Add `--focus tech` (or another preset) to restrict extraction |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this recipe

---
