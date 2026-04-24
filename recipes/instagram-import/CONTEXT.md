# CONTEXT.md — Instagram Import

## Purpose

Parses Instagram GDPR/data-download export archives and imports DM conversations, post comments, and post captions into Open Brain as embedded thoughts. Operates as a one-shot CLI script against a local export directory rather than via the Instagram API.

## Responsibility Boundaries

- **Owns**: Locating and parsing the `your_instagram_activity` folder structure, encoding repair, batching, content fingerprinting, embedding generation, and upserting thoughts via `upsert_thought` RPC
- **Delegates to**: OpenRouter API for embedding generation; Supabase `upsert_thought` RPC for deduplication and persistence
- **Does not handle**: Instagram API access, incremental sync, media/image content, or story data

## Key Concepts

- **Content fingerprint**: SHA-256 of normalised (lowercased, whitespace-collapsed) content passed as `content_fingerprint` metadata to prevent re-importing identical items across runs
- **`upsert_thought` RPC**: The single Supabase function used for all writes; handles insert-or-update logic on the `thoughts` table

## Non-Obvious Details

- **Meta encoding bug**: Instagram exports encode UTF-8 text as latin1, producing mojibake for non-ASCII characters (e.g., emoji, accented letters). `fixMetaEncoding` repairs this with `Buffer.from(text, "latin1").toString("utf-8")`. Without this step, all non-ASCII content in messages, comments, and captions is corrupt.
- **Export directory discovery**: The script walks up to 3 levels deep to find `your_instagram_activity`, accommodating varying zip-extraction layouts without requiring an exact path argument.
- **Conversation capping**: DM threads are capped at 200 messages per conversation and threads with fewer than 3 messages are skipped entirely to avoid noise.
- **Comment and post batching**: Comments are grouped into batches of 50 per thought; post captions into batches of 30. This keeps individual thought content within useful embedding limits while avoiding one thought per item at scale.
- **Embedding truncation**: Text longer than 8,000 characters is truncated before being sent to the embedding model. Thoughts stored in Supabase are independently capped at 30,000 characters.
- **Comment JSON schema variants**: The comment parser handles two different key layouts (`comments_media_comments` array vs. raw array, and `string_map_data.Comment` vs. `string_map_data.comment`) to tolerate variation across Instagram export versions.
- **`--types` flag**: Allows selective import of `messages`, `comments`, and/or `posts` in a single run, enabling targeted re-imports without duplicating already-imported types (fingerprinting prevents exact duplicates regardless, but selective import reduces API calls).
- **Thoughts are tagged** with `source_type: "instagram_import"`, `sensitivity_tier: "personal"`, `importance: 2`, and `quality_score: 40` at upsert time — these defaults reflect the expected lower signal density of social media content vs. deliberate journal entries.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Idempotent PR comments)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, upsert_thought RPC)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), upsert_thought RPC)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), upsert_thought RPC)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, upsert_thought RPC)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Capture integration, upsert_thought RPC)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), re-extraction idempotency)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, prepareThoughtPayload, upsert_thought RPC)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Import and Export Pipelines domain (Meta latin1/UTF-8 encoding repair, Portable Bundle)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Sync log)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Sync log, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprint (SHA-256 deduplication), Duplicate row)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Day-hash dedup via sync log)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Content fingerprint deduplication)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Content fingerprint for deduplication)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint (SHA-256 deduplication))
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Session-scoped deduplication)
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (Meta latin1/UTF-8 encoding repair, Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Dual deduplication (sync log + content fingerprint))
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Local sync log deduplication)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, upsert_thought RPC)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (captureThought pipeline, upsert_thought RPC)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (captureThought pipeline, upsert_thought RPC)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), Meta latin1/UTF-8 encoding repair)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Content fingerprinting)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), idempotent schema migration)
- **[server](../../server/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Two-step capture (upsert + embedding patch), upsert_thought RPC, upsert_thought RPC (deduplication-aware insert))
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Open Brain deduplication workflow)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Meta latin1/UTF-8 encoding repair)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Meta latin1/UTF-8 encoding repair, export bundles)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), Meta latin1/UTF-8 encoding repair)
