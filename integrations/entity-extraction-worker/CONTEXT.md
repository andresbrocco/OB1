# CONTEXT.md — Entity Extraction Worker

## Purpose

An async Deno Edge Function that drains a database-backed queue (`entity_extraction_queue`) by calling an LLM to extract named entities and relationships from thought content, then writing results into the knowledge graph tables (`entities`, `edges`, `thought_entities`). It is designed to run on a cron schedule rather than in-line with thought capture, decoupling extraction latency from the write path.

## Responsibility Boundaries

- **Owns**: Queue drain logic, LLM prompt construction and response parsing, entity/edge upsert, `thought_entities` link lifecycle, per-invocation cost/time budget enforcement.
- **Delegates to**: `schemas/knowledge-graph` (table definitions for `entity_extraction_queue`, `entities`, `edges`, `thought_entities`); `schemas/enhanced-thoughts` (enhanced columns on `thoughts`); LLM providers (OpenRouter → OpenAI → Anthropic fallback chain).
- **Does not handle**: Enqueueing thoughts into the queue (that is done by a database trigger in the knowledge-graph schema), thought classification/enrichment (handled by the capture path), or embedding generation.

## Key Concepts

- **entity_extraction_queue**: A Postgres table acting as the work queue. Thoughts are enqueued by a database trigger on content change. The worker claims rows atomically by transitioning `status` from `pending` → `processing` using a conditional update with `.eq("status", "pending")` to prevent double-processing by concurrent invocations.
- **support_count on edges**: When an edge already exists, the worker increments its `support_count` rather than replacing it. This accumulates evidence across multiple thoughts that mention the same relationship.
- **Symmetric relation canonicalization**: For relations like `co_occurs_with` and `related_to`, the edge is always stored with the lower entity ID as `from_entity_id` to prevent duplicate rows for the same undirected relationship.
- **Re-extraction idempotency**: When a thought's content changes, the trigger re-enqueues it. The worker deletes prior `thought_entities` links scoped to `source='entity_worker'` before writing new ones, so stale entity links from the previous version of the content are cleaned up without affecting links written by other sources.
- **ExtractionCostCapError**: A module-scoped call counter (`llmCallCount`) enforces a per-container ceiling (`ENTITY_EXTRACTION_MAX_CALLS`, default 10,000). The counter resets on cold start, not across deploys — designed to block a runaway hot container, not to enforce a global quota.

## Non-Obvious Details

- **Wall-clock budget**: The handler stops claiming new items at 140 seconds (10 seconds before Supabase Edge Functions' hard 150s kill). Unclaimed items that were already claimed but not yet processed are released back to `pending` so they are not stuck in `processing`.
- **Fetch timeout**: Every outbound LLM call is wrapped in an `AbortController` with a default 60-second timeout (`FETCH_TIMEOUT_MS`). Without this, a stalled upstream can consume the entire 150s wall-clock budget on a single call.
- **Prompt injection defense**: Thought content is wrapped in `<thought_content>` XML-like delimiters, with any literal occurrences of those tags escaped before insertion. The system prompt explicitly instructs the LLM to return empty arrays if it detects an injection attempt.
- **System-generated thoughts are skipped**: Thoughts with `metadata.generated_by` set are silently marked `skipped` in the queue rather than extracted — prevents the worker from building circular entity relationships from AI-authored content.
- **`_shared/` is a copy, not a symlink**: The `_shared/config.ts` and `_shared/helpers.ts` files contain the same shared utilities used by other edge functions in this integration layer. Each edge function embeds its own copy because Supabase Edge Functions cannot share code across function directories at deploy time.
- **dry_run mode**: Passing `?dry_run=true` runs the full LLM extraction but skips all DB writes and returns the would-be entities/relationships in the response body. Useful for validating extraction quality without mutating state.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, PR quota enforcement, wall-clock budget)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, ExtractionCostCapError, wall-clock budget)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Post-search filter extraction, entity_extraction_queue)
- **[integrations](../CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, ExtractionCostCapError, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, _enrichment_status, entity_extraction_queue)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Two-Prompt Extraction Sequence, entity_extraction_queue)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Sync log, re-extraction idempotency)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, ExtractionCostCapError, entity_extraction_queue)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Sync log, Two-layer dedup, re-extraction idempotency)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, support_count, symmetric relation canonicalization)
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, re-extraction idempotency)
- **[recipes/google-activity-import](../../recipes/google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, re-extraction idempotency)
- **[recipes/grok-export-import](../../recipes/grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, re-extraction idempotency)
- **[recipes/instagram-import](../../recipes/instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), re-extraction idempotency)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, re-extraction idempotency)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, re-extraction idempotency)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Session-scoped deduplication, re-extraction idempotency)
- **[recipes/ob-graph](../../recipes/ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, support_count, symmetric relation canonicalization, traverse_graph (outgoing recursive CTE))
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), re-extraction idempotency)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Local sync log deduplication, re-extraction idempotency)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Pending person confirmation, Three-pass person resolution, entity_extraction_queue)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, ExtractionCostCapError, entity_extraction_queue)
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, Hard cost cap with proactive parallelism clamping, wall-clock budget)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, In-memory rate limiter with cold-start reset, wall-clock budget)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, in-memory sliding-window rate limiter, wall-clock budget)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (derived_from edges, support_count, symmetric relation canonicalization)
- **[recipes/x-twitter-import](../../recipes/x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, re-extraction idempotency)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, ExtractionCostCapError, entity_extraction_queue)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Depends on for Extends the thoughts table with structured classification columns and installs utility RPCs for full-text search, aggregate statistics, and thought-connection discovery
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, ExtractionCostCapError, Thought-entity mention role and evidence, entity_extraction_queue)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), support_count, support_count evidence accumulation, symmetric relation canonicalization)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, re-extraction idempotency)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, ExtractionCostCapError, wall-clock budget)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Thread extraction, entity_extraction_queue)
