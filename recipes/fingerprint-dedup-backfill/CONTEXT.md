# CONTEXT.md — Fingerprint Dedup Backfill

## Purpose

A two-script Node.js toolset for retroactively adding `content_fingerprint` values to existing `thoughts` rows that predate the `content-fingerprint-dedup` primitive, and for safely removing duplicate rows discovered during that process.

## Responsibility Boundaries

- **Owns**: Backfilling `content_fingerprint` on NULL rows; identifying and optionally deleting NULL-fingerprint rows whose content is already represented by a fingerprinted canonical row
- **Delegates to**: Supabase REST API for all reads, writes, and deletes; the `content-fingerprint-dedup` primitive for the canonical fingerprint normalization contract
- **Does not handle**: Deduplication of rows that already have a fingerprint set; schema migration; ongoing real-time dedup (that belongs to the trigger defined in `content-fingerprint-dedup`)

## Key Concepts

- **Content fingerprint**: A SHA-256 hex digest of normalized thought content. Normalization trims whitespace, lowercases, strips trailing punctuation, strips possessives, and strips a trailing `s` from words longer than 3 characters. This JavaScript implementation must stay byte-for-byte identical to the `normalize_for_fingerprint(text)` SQL function in the `content-fingerprint-dedup` primitive.
- **Orphan row**: A NULL-fingerprint row whose computed fingerprint does not match any existing fingerprinted row. It is safe to backfill rather than delete.
- **Duplicate row**: A NULL-fingerprint row whose computed fingerprint already exists in the table. `delete-duplicates.mjs` removes these; `backfill-fingerprints.mjs` skips them (it receives a 409/23505 conflict and counts them separately).
- **Cursor-based resumability**: Both scripts persist a `cursorId` (the highest `id` processed) to a local JSON state file after each batch, so an interrupted run restarts from the last committed batch rather than the beginning.

## Non-Obvious Details

- **Run order matters**: `delete-duplicates.mjs` should run before `backfill-fingerprints.mjs` on a large database. Running backfill first will assign fingerprints to what would have been duplicates, causing a unique-constraint conflict on the column instead of a clean delete.
- **`delete-duplicates.mjs` is report-only by default**: It must be invoked with `--delete` to perform any destructive action. Without the flag it prints a count of rows that would be removed.
- **409/23505 conflicts in backfill are non-fatal**: The backfill script treats unique-constraint violations as "already handled" duplicates and counts them separately rather than as errors — this is intentional and not a silent failure.
- **State files are ephemeral**: Both scripts delete their state file on successful completion (`backfill-state.json` and `cleanup-state.json`). If a state file exists at startup, the run resumes from that checkpoint rather than starting over.
- **Concurrency is intentionally throttled**: `backfill-fingerprints.mjs` uses a concurrency of 20 PATCH requests per chunk with a 150 ms inter-batch delay to avoid overwhelming the Supabase REST API.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Idempotent PR comments)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Orphan row, ephemeral IDs)
- **[extensions/family-calendar](../../extensions/family-calendar/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (NULL family_member_id for household-wide events, Orphan row)
- **[extensions/home-maintenance](../../extensions/home-maintenance/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Orphan row, frequency_days=NULL for one-time tasks)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, re-extraction idempotency)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Sync log)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Sync log, Two-layer dedup)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Day-hash dedup via sync log, Duplicate row)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprint deduplication, Duplicate row)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (--redo flag, Cursor-based resumability)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprint (SHA-256 deduplication), Duplicate row)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprint for deduplication, Duplicate row)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint, Duplicate row)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Session-scoped deduplication)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Dual deduplication (sync log + content fingerprint), Duplicate row)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Local sync log deduplication)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Cursor-based resumable state)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Phase toggles)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Resume-safe JSONL state log)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Checkpoint + Entry Separation, Cursor-based resumability)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprinting, Duplicate row)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, idempotent schema migration)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Async queue with content-addressed re-queue, Cursor-based resumability)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Orphan row, Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Open Brain deduplication workflow)
