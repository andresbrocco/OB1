# CONTEXT.md — Typed Edge Classifier

## Purpose

Populates `public.thought_edges` with typed semantic reasoning relations between pairs of thoughts in an Open Brain instance. Given candidate thought pairs, it classifies the directional relationship (e.g., `supports`, `contradicts`, `supersedes`, `depends_on`) and inserts edges with confidence scores, classifier provenance, and optional temporal bounds.

## Responsibility Boundaries

- **Owns**: Candidate pair selection from `thought_entities` overlap, two-stage LLM classification pipeline, cost-cap enforcement, edge insertion via `thought_edges_upsert` RPC, and optional mirroring to `thoughts.supersedes`
- **Delegates to**: `schemas/typed-reasoning-edges` for the `thought_edges` table schema and `thought_edges_upsert` RPC; `schemas/entity-extraction` for the `thought_entities` table used in candidate sampling
- **Does not handle**: Embedding-based similarity, entity extraction itself, or retrieval queries over the resulting edge graph

## Key Concepts

- **Hybrid filter+classify pipeline**: Haiku runs a cheap yes/no pre-filter on each candidate pair; only pairs that pass advance to Opus for full classification with the typed relation vocabulary. This is the primary cost-reduction mechanism (~10–20x cheaper filter leg at 20–40% pass rates).
- **Typed relation vocabulary**: Six specific relation types — `supports`, `contradicts`, `evolved_into`, `supersedes`, `depends_on`, `related_to` — plus `none`. Each has strict inclusion/exclusion criteria embedded in the Opus system prompt. The set must match the `CHECK` constraint in `schemas/typed-reasoning-edges/schema.sql`.
- **Edge direction**: The classifier outputs `A_to_B`, `B_to_A`, or `symmetric`. For symmetric relations, from/to UUIDs are lexically sorted so re-runs produce the same stable key and hit the upsert's `ON CONFLICT` instead of creating duplicates.
- **Cost cap**: `--max-cost-usd` is enforced as a hard bound. The proactive parallelism clamp (`processInChunks`) computes worst-case per-pair spend (Haiku filter + Opus classify legs combined) and reduces chunk size as the cap approaches, bounding overshoot to at most one worst-case pair.
- **Idempotent upsert**: Inserts go through the `thought_edges_upsert` RPC (not a plain POST), which uses `ON CONFLICT DO UPDATE` to increment `support_count` and refresh `valid_until` on repeat classifications rather than raising a uniqueness error.

## Non-Obvious Details

- **`related_to` is skippable for reclassification**: The already-classified check filters `relation != 'related_to'`, so pairs previously tagged with the fallback label remain eligible for reclassification on subsequent runs. Pairs with stronger labels are permanently skipped.
- **`--mirror-supersedes` is best-effort, not atomic**: When enabled, a second `PATCH` updates `thoughts.supersedes` after the edge is inserted. If that PATCH fails (e.g., the `schemas/provenance-chains` schema is not applied), a warning is logged but the edge remains. Because the already-classified check short-circuits re-runs, a failed mirror will not be retried automatically; manual SQL reconciliation is required.
- **Unknown model pricing causes startup refusal**: If `--max-cost-usd` is set (the default) and a model is not in the internal `PRICING` map, the script refuses to start. Pass `--no-cost-cap` to acknowledge that the cap cannot be enforced. This prevents silent cost-cap bypass from model name drift.
- **Candidate sampling requires `thought_entities`**: The default sampling strategy queries `thought_entities` for pairs with shared entity overlap. If that table does not exist, the script errors out and requires an explicit `--pair UUID_A,UUID_B` argument.
- **PostgREST result order is not guaranteed**: `fetchThoughts` uses a `Map<id, row>` lookup rather than positional indexing so that A and B cannot silently swap, which would corrupt directional edges.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Hard cost cap with proactive parallelism clamping, PR quota enforcement)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, Hard cost cap with proactive parallelism clamping)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, Hard cost cap with proactive parallelism clamping)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, Hard cost cap with proactive parallelism clamping, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Sync log)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Sync log, Two-layer dedup)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), co_occurs_with exclusion)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Session-scoped deduplication)
- **[recipes/ob-graph](../ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Local sync log deduplication)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Schema-aware routing)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, LLM classification prompt with importance/confidence calibration)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Hard cost cap with proactive parallelism clamping, In-memory rate limiter with cold-start reset)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Hard cost cap with proactive parallelism clamping, in-memory sliding-window rate limiter)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Provides classify-edges.mjs (CLI script) consumed by this module
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Synthesizer catalogue)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), derived_from edges)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Idempotent upsert via thought_edges_upsert RPC)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to))
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, idempotent schema migration)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Depends on for Knowledge graph schema layer with async extraction queue and consolidation audit trail for automatic entity and relationship extraction from thoughts
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Depends on for Database schema extension adding typed semantic reasoning edges between thoughts and temporal validity decay to entity edges
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Open Brain deduplication workflow)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, Hard cost cap with proactive parallelism clamping)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Mode classification)
