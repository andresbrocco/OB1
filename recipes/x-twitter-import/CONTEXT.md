# CONTEXT.md — X/Twitter Import

## Purpose

Parses a Twitter/X data export archive and imports its contents into Open Brain as searchable thoughts with embeddings. Handles three content types from the export: original tweets, direct message conversations, and Grok AI chat histories.

## Responsibility Boundaries

- **Owns**: Parsing Twitter's JS-wrapped export format, batching/structuring content into thought-sized units, generating embeddings via OpenRouter, and upserting to Supabase via the `upsert_thought` RPC
- **Delegates to**: OpenRouter API for embedding generation, Supabase `upsert_thought` RPC for deduplication and persistence
- **Does not handle**: Authentication with Twitter's API, live streaming, or incremental sync — this is a one-time archive import only

## Key Concepts

- **Twitter JS export format**: Twitter's data export wraps JSON arrays in a JavaScript assignment (`window.YTD.tweets.part0 = [...]`). The parser strips this prefix by finding the first `[` character rather than parsing valid JSON directly.
- **Tweet batching**: Individual tweets are too short to embed meaningfully. The script groups tweets into batches of 20 (sorted chronologically) and imports each batch as a single thought, preserving the date range as context.
- **Content fingerprinting**: Each thought is hashed (SHA-256 on normalized text) before upsert, enabling the `upsert_thought` RPC to detect and skip duplicate content on re-runs.

## Non-Obvious Details

- Retweets (`RT @`) are filtered out entirely — only original tweets are imported.
- DM conversations with fewer than 3 messages are skipped as too sparse to be useful.
- The `--types` flag controls which content types are processed (`tweets`, `dms`, `grok`); all three are enabled by default.
- The export's `data/` subdirectory is auto-detected; if absent, the root of the provided path is used as a fallback.
- Content is truncated at 30,000 characters per thought and at 8,000 characters before embedding to stay within model context limits.
- All thoughts are inserted with `type: "reference"`, `importance: 2`, and `quality_score: 40` — fixed values, not derived from engagement metrics.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Idempotent PR comments)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, re-extraction idempotency)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Import and Export Pipelines domain (Portable Bundle, Tweet batching, Twitter JS export format)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Sync log)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Sync log, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprinting, Duplicate row)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Day-hash dedup via sync log)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Content fingerprinting)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Content fingerprinting)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Content fingerprinting)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprinting)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Session-scoped deduplication)
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (Multi-format input (stdin, positional args, .txt, .jsonl), Tweet batching, Twitter JS export format)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Dual deduplication (sync log + content fingerprint))
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Local sync log deduplication)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), Tweet batching, Twitter JS export format)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, idempotent schema migration)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Open Brain deduplication workflow)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Tweet batching, Twitter JS export format)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Tweet batching, Twitter JS export format, export bundles)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), Tweet batching, Twitter JS export format)
