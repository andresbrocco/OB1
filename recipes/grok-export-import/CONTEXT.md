# CONTEXT.md — Grok Export Import

## Purpose

Imports xAI Grok conversation history exports (JSON format) into Open Brain as searchable `thoughts` with vector embeddings. Each Grok conversation becomes a single thought record, enabling semantic search over past AI interactions.

## Responsibility Boundaries

- **Owns**: Parsing Grok export JSON, normalizing conversation structure into a flat transcript, generating embeddings via OpenRouter, and upserting records into the Open Brain `thoughts` table.
- **Delegates to**: OpenRouter API for embedding generation; Supabase `upsert_thought` RPC for deduplication and persistence.
- **Does not handle**: Scheduling, incremental sync, or tracking previously imported exports between runs (deduplication is handled at the `upsert_thought` RPC level via content fingerprint).

## Key Concepts

- **MongoDB-style dates**: Grok exports encode timestamps using MongoDB Extended JSON format (`{ "$date": { "$numberLong": "..." } }`). The `parseMongoDate` function handles both this format and plain ISO strings.
- **Content fingerprint**: A SHA-256 hash of the normalized (lowercased, whitespace-collapsed) conversation text is computed and passed to `upsert_thought` to prevent duplicate imports across runs.
- **Conversation normalization**: Grok exports use inconsistent field names across export versions (`conversation`, `messages`, `responses`; `sender`, `role`; `message`, `text`, `content`). The `normalizeConversation` function probes multiple field names to accommodate schema variation.
- **Transcript format**: All messages are assembled into a single `USER: ... / ASSISTANT: ...` transcript string, which becomes the `content` of the thought. Content is hard-truncated at 30,000 characters and embeddings are computed on the first 8,000 characters.

## Non-Obvious Details

- The script uses `SUPABASE_SERVICE_ROLE_KEY` (not the anon key), which bypasses Row Level Security. This is required to write thoughts on behalf of the owner without an authenticated session.
- The `--skip N` and `--limit N` flags allow resuming a partial import if the script is interrupted, since there is no persistent import-state file.
- Conversations with fewer than 100 characters of content after normalization are silently skipped.
- The `type` field on inserted thoughts is hardcoded to `"reference"` and `importance` to `3`, meaning these imports are treated as lower-priority reference material rather than first-person memories.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Idempotent PR comments)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, re-extraction idempotency)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Import and Export Pipelines domain (MongoDB-style date parsing, Portable Bundle)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Conversation tree / branch resolution, Session splitting, Transcript assembly)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Sync log, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprint deduplication, Duplicate row)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Day-hash dedup via sync log)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Content fingerprint deduplication)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Content fingerprint for deduplication)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint deduplication)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Topic shift detection, Transcript assembly)
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (MongoDB-style date parsing, Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Dual deduplication (sync log + content fingerprint))
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Speaker Consolidation, Thread, Transcript assembly)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Transcript assembly, Two-sheet import (Conversations + Memory))
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Thread eligibility (content-weight gating), Transcript assembly)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Thread eligibility gate, Transcript assembly)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Session Versioning, Transcript assembly)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Content fingerprinting)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, idempotent schema migration)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Open Brain deduplication workflow)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), MongoDB-style date parsing)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, MongoDB-style date parsing, export bundles)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Speaker Consolidation, Transcript assembly)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), MongoDB-style date parsing)
