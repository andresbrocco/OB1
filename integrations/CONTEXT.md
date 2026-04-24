# CONTEXT.md — Integrations

## Purpose

External connection points that bring data into Open Brain or replace its infrastructure layer. Each sub-folder is a standalone, deployable contribution: message-capture bots (Slack, Discord), async background workers that process internal queues, and alternative deployment targets (Kubernetes self-hosting).

## Responsibility Boundaries

- **Owns**: The code that bridges an external system or runtime to the Open Brain data layer (Supabase or direct PostgreSQL)
- **Delegates to**: `schemas/` for table definitions the workers depend on; `primitives/` for deployment patterns; the AI client for MCP tool invocations
- **Does not handle**: Core `thoughts` table schema changes, client-side prompt packs (those belong in `skills/`), or multi-step orchestration (those belong in `recipes/`)

## Key Concepts

- **Capture integrations** (slack-capture, discord-capture): Webhook or bot receivers that write raw messages as thoughts. They mirror each other structurally — Discord follows the Slack capture pattern.
- **Async worker** (entity-extraction-worker): A Supabase Edge Function that drains the `entity_extraction_queue` table. It does not run continuously; it is invoked by a cron or manual trigger, claims a batch of pending rows atomically, calls an LLM, and writes to the `entities` / `edges` / `thought_entities` tables. The worker has a module-scoped call counter (`llmCallCount`) that acts as a per-cold-start cost cap, deliberately resetting on each container boot.
- **Kubernetes deployment**: A self-hosted variant of the core MCP server that replaces the Supabase client with a direct PostgreSQL connection pool. It is structurally distinct from the Edge Function pattern used everywhere else in the repo — it runs as a persistent Hono HTTP server inside a container rather than a serverless function.
- **`_shared/config.ts`**: A shared constants and type module used by entity-extraction-worker. Defines the 0–6 importance scale, sensitivity tier patterns (regex-based PII detection), allowed thought types, and the multi-provider classifier model fallback order (OpenRouter → OpenAI → Anthropic).

## Non-Obvious Details

- The entity-extraction-worker uses optimistic locking for queue claims: it selects pending rows then updates them with `.eq("status", "pending")` so concurrent invocations don't double-process. Rows that were claimed but not processed before a wall-clock or cost-cap trip are released back to `"pending"` before the function returns.
- The worker skips thoughts whose `metadata.generated_by` field is set, preventing LLM-generated thoughts from being re-extracted.
- On re-extraction (triggered when thought content changes), the worker deletes prior `thought_entities` rows scoped to `source='entity_worker'` before writing new ones, so stale entity links don't accumulate.
- Symmetric relations (`co_occurs_with`, `related_to`) are canonically ordered by entity ID (`min(id), max(id)`) before insertion to prevent duplicate edges with swapped directions.
- The Kubernetes deployment intentionally does not use `StdioServerTransport` or `claude_desktop_config.json`; it exposes an HTTP endpoint via Hono + `StreamableHTTPTransport`, consistent with the repo's remote-MCP-only constraint.
- `_shared/config.ts` is named for the "Enhanced MCP integration" context, not for `entity-extraction-worker` specifically — it is a shared module that could be reused by other workers in this directory.

## Related Modules

- **[.github](../.github/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, PR quota enforcement)
- **[.github/workflows](../.github/workflows/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, Contributor trust and quota policy)
- **[dashboards](../dashboards/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP JSON-RPC proxy connection pattern (SvelteKit dashboard))
- **[dashboards/open-brain-dashboard](../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP proxy route)
- **[dashboards/open-brain-dashboard-next/app/api](../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), Capture integration)
- **[dashboards/open-brain-dashboard-next/components](../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Capture integration, Dry-run two-phase ingestion, Ingestion modes (auto/single/extract))
- **[dashboards/open-brain-dashboard-next/lib](../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Capture integration)
- **[dashboards/open-brain-dashboard/src](../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP credential server-vs-public pattern)
- **[dashboards/open-brain-dashboard/src/lib](../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP text-response parsing)
- **[dashboards/open-brain-dashboard/src/routes](../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, entity_extraction_queue)
- **[docs](../docs/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP tool context overhead)
- **[extensions](../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, Per-request MCP server instantiation)
- **[extensions/family-calendar](../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Kubernetes self-hosted MCP server)
- **[extensions/household-knowledge](../extensions/household-knowledge/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Shared config with sensitivity tiers, details JSONB freeform metadata field)
- **[extensions/meal-planning](../extensions/meal-planning/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, Shared config with sensitivity tiers)
- **[extensions/professional-crm](../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, Stateless MCP transport)
- **[integrations/entity-extraction-worker](entity-extraction-worker/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, ExtractionCostCapError, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (_enrichment_status, entity_extraction_queue)
- **[integrations/kubernetes-deployment](kubernetes-deployment/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Kubernetes self-hosted MCP server, Supabase replacement pattern)
- **[integrations/kubernetes-deployment/k8s](kubernetes-deployment/k8s/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Kubernetes self-hosted MCP server, Supabase schema parity, hostPath volume)
- **[recipes](../recipes/CONTEXT.md)** — Provides entity-extraction-worker (Edge Function), kubernetes-deployment (Hono HTTP server), slack-capture, ... consumed by this module
- **[recipes/bring-your-own-context](../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Two-Prompt Extraction Sequence, entity_extraction_queue)
- **[recipes/claudeception](../recipes/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, entity_extraction_queue)
- **[recipes/email-history-import](../recipes/email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Capture integration, Ingestion modes)
- **[recipes/entity-wiki](../recipes/entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Symmetric relation canonical ordering, co_occurs_with exclusion)
- **[recipes/google-activity-import](../recipes/google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Capture integration, Thought prefix format on insert)
- **[recipes/instagram-import](../recipes/instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Capture integration, upsert_thought RPC)
- **[recipes/life-engine](../recipes/life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, entity_extraction_queue)
- **[recipes/ob-graph](../recipes/ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Symmetric relation canonical ordering, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/obsidian-vault-import](../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Capture integration)
- **[recipes/perplexity-conversation-import](../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Shared config with sensitivity tiers)
- **[recipes/repo-learning-coach/src](../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Kubernetes self-hosted MCP server)
- **[recipes/repo-learning-coach/src/lib](../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Kubernetes self-hosted MCP server)
- **[recipes/schema-aware-routing](../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Three-pass person resolution, entity_extraction_queue)
- **[recipes/thought-enrichment](../recipes/thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, entity_extraction_queue)
- **[recipes/typed-edge-classifier](../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, Hard cost cap with proactive parallelism clamping)
- **[recipes/vercel-neon-telegram](../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, In-memory rate limiter with cold-start reset)
- **[recipes/vercel-neon-telegram/src](../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, in-memory sliding-window rate limiter)
- **[recipes/vercel-neon-telegram/src/app/api](../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request)
- **[recipes/vercel-neon-telegram/src/lib](../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Shared config with sensitivity tiers, ThoughtMetadata)
- **[recipes/wiki-synthesis/scripts](../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Symmetric relation canonical ordering, derived_from edges)
- **[schemas](../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, entity_extraction_queue)
- **[schemas/enhanced-thoughts](../schemas/enhanced-thoughts/CONTEXT.md)** — Depends on for Extends the thoughts table with structured classification columns and installs utility RPCs for full-text search, aggregate statistics, and thought-connection discovery
- **[schemas/entity-extraction](../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, entity_extraction_queue)
- **[schemas/typed-reasoning-edges](../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), Symmetric relation canonical ordering, support_count evidence accumulation)
- **[server](../server/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, Kubernetes self-hosted MCP server, MCP tool registration, StreamableHTTPTransport)
- **[skills](../skills/CONTEXT.md)** — Provides entity-extraction-worker (Edge Function), kubernetes-deployment (Hono HTTP server), slack-capture, ... consumed by this module
- **[skills/claudeception](../skills/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, entity_extraction_queue)
- **[skills/financial-model-review](../skills/financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, Shared config with sensitivity tiers)
- **[skills/heavy-file-ingestion](../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, Cost tier escalation)
- **[skills/n-agentic-harnesses](../skills/n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, Shared config with sensitivity tiers)
- **[skills/panning-for-gold](../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Thread extraction, entity_extraction_queue)
- **[skills/weekly-signal-diff](../skills/weekly-signal-diff/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Kubernetes self-hosted MCP server, Starter universe bootstrap)
