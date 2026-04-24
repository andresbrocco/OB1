# CONTEXT.md — Google Activity Import

## Purpose

Converts a Google Takeout "My Activity" export into a set of searchable, first-person thoughts in Open Brain. Raw activity entries (searches, map queries, email subjects, browser history) are noise-filtered, grouped by calendar day and Google service category, then passed through an LLM that distills each day-group into 1–3 standalone thoughts before embedding and inserting them into the `thoughts` table.

## Responsibility Boundaries

- **Owns**: Takeout file discovery, per-category noise filtering, day-level grouping, LLM summarization, embedding generation, Supabase insertion, and idempotency tracking via a local sync log.
- **Delegates to**: OpenRouter for both summarization (gpt-4o-mini) and embeddings (text-embedding-3-small); Supabase REST API for persistence.
- **Does not handle**: OAuth or Google API calls — input is always a local Takeout export, never a live API session.

## Key Concepts

- **High-value categories**: Only six Google service categories are processed by default (`Search`, `Gmail`, `Maps`, `YouTube`, `Chrome`, `Gemini Apps`). Low-signal categories (Ads, Assistant, etc.) are omitted entirely.
- **Per-category noise rules**: Each category applies distinct filter logic. Maps drops passive entries (`Visited`, `Viewed`, `Opened`), keeping only searches and navigation. YouTube drops short `Watched` titles. Chrome requires a minimum title length. These rules are hardcoded in `filterActivities`.
- **Day-hash dedup**: The sync log (`google-activity-sync-log.json`, written to the working directory) records a SHA-256 hash of each `category:date` pair's filtered entries. Re-running the script skips any day whose hash matches, making imports idempotent and safe to interrupt and resume.
- **Raw mode**: `--raw` skips LLM summarization entirely and inserts the concatenated day-group text as a single thought. Useful for offline testing or when OpenRouter is unavailable.
- **Thought format on insert**: Each thought is stored with a `[Google {Category}: {YYYY-MM-DD}]` prefix prepended to the LLM-generated text, and metadata fields `source`, `google_category`, `google_date`, and `entry_count`.

## Non-Obvious Details

- The sync log is written to the **current working directory** at runtime, not to the recipe folder. Running the script from different directories will produce separate, non-shared sync logs.
- Entries per day are capped before LLM submission: 30 for Maps, 100 for all other categories. Entries beyond the cap are silently dropped before summarization.
- Day-level dedup only updates the sync log if **all** thoughts for that day ingested successfully. A partial failure leaves the day unlogged so it will be retried on the next run.
- The summarization prompt instructs the model to return `{"thoughts": []}` for trivial days and explicitly prefers returning empty over over-capturing. Days marked trivial are still written to the sync log so they are not re-evaluated on subsequent runs.
- API cost is estimated and printed at the end of each run (gpt-4o-mini input/output rates plus embedding token cost), but this is a rough estimate, not an exact measurement.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Idempotent PR comments)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, Thought prefix format on insert)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), Thought prefix format on insert)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), Thought prefix format on insert)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Thought prefix format on insert)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Capture integration, Thought prefix format on insert)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, re-extraction idempotency)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, Thought prefix format on insert, prepareThoughtPayload)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Sync log)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Sync log, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Day-hash dedup via sync log, Duplicate row)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Day-hash dedup via sync log)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Day-hash dedup via sync log)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Day-hash dedup via sync log)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Day-hash dedup via sync log)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Session-scoped deduplication)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Dual deduplication (sync log + content fingerprint))
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Gold-Found, High-value categories, Panning, Per-category noise filtering)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Local sync log deduplication)
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Skip rules)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, High-value categories, Per-category noise filtering)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, Thought prefix format on insert)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Thought prefix format on insert, captureThought pipeline)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Thought prefix format on insert, captureThought pipeline)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Thread eligibility gate)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Day-hash dedup via sync log)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, idempotent schema migration)
- **[server](../../server/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Thought prefix format on insert, Two-step capture (upsert + embedding patch), upsert_thought RPC (deduplication-aware insert))
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Open Brain deduplication workflow)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Panning / Gold-Found, Per-category noise filtering)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Signal diff vs digest)
