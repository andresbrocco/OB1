# CONTEXT.md — Vercel + Neon + Telegram

## Purpose

An alternative Open Brain deployment architecture that substitutes the default Cloudflare + Supabase + Slack stack with Vercel serverless functions, Neon Postgres (with pgvector), and Telegram. It preserves the same `thoughts` table schema and MCP interface so the core memory system is interchangeable. The recipe bundles thought capture, semantic search, a REST API, and an MCP endpoint in a single Next.js app.

## Responsibility Boundaries

- **Owns**: Thought ingestion (embedding + metadata extraction + insert), semantic vector search, MCP server definition (`capture_thought`, `search_thoughts`, `list_thoughts` tools), Telegram bot webhook handling, REST capture endpoint, access-key authentication, and in-memory rate limiting.
- **Delegates to**: OpenAI API for embedding generation and metadata extraction; Neon Postgres for vector storage and the `match_thoughts` SQL function.
- **Does not handle**: User account management, multi-tenant isolation, persistent session state, or Telegram media beyond captioned photos.

## Key Concepts

- **Stateless MCP transport**: The `/api/mcp` route creates a fresh `McpServer` + `WebStandardStreamableHTTPServerTransport` per request. There is no session negotiation — `sessionIdGenerator` is set to `undefined` and `DELETE`/`HEAD` methods exist only for protocol compliance.
- **Parallel capture pipeline**: `captureThought` fires embedding generation and metadata extraction concurrently via `Promise.all`, then inserts once both resolve.
- **ThoughtType taxonomy**: Seven discrete types (`observation`, `task`, `idea`, `reference`, `person_note`, `decision`, `meeting_note`) are enforced at the Zod schema layer and propagated into Postgres `metadata` JSONB.
- **In-memory rate limiter**: `rate-limit.ts` tracks request timestamps in a module-level array. It resets on every cold start, making it suitable only for personal/single-user deployments; it does not coordinate across concurrent Vercel instances.

## Non-Obvious Details

- **Auth accepts three credential locations**: `x-brain-key` header, `Authorization: Bearer <key>`, or a `?key=` query parameter. The query parameter path exists specifically to support AI clients (e.g., ChatGPT) that cannot send custom headers.
- **Telegram webhook secret is separate from `BRAIN_ACCESS_KEY`**: The Telegram endpoint validates `x-telegram-bot-api-secret-token` against `TELEGRAM_WEBHOOK_SECRET`, not the shared brain key. The bot channel bypasses the normal auth layer entirely.
- **`listThoughts` uses dynamic SQL with positional params** while `insertThought` and `searchThoughts` use the tagged-template `sql` client. The inconsistency is intentional — `listThoughts` needs conditional WHERE clauses that the tagged-template API cannot compose cleanly.
- **Rate limiter is only applied to the MCP and REST capture endpoints**, not to the Telegram bot handler. A Telegram flood would not be throttled by `checkRateLimit`.
- **`maxDuration` export**: The Telegram route sets `maxDuration = 15` and the MCP route sets `maxDuration = 30` — these are Vercel-specific route-level execution timeout overrides, not standard Next.js.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (In-memory rate limiter with cold-start reset, PR quota enforcement)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Parallel capture pipeline)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), Stateless MCP transport)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, Stateless MCP transport)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType taxonomy, kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), Parallel capture pipeline)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), Parallel capture pipeline)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Parallel capture pipeline)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Stateless MCP transport)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Stateless MCP transport)
- **[docs](../../docs/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP tool context overhead, Stateless MCP transport)
- **[extensions](../../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, Stateless MCP transport)
- **[extensions/family-calendar](../../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Stateless MCP transport)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Stateless MCP transport)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request stateless MCP server instantiation, Stateless MCP transport)
- **[extensions/professional-crm](../../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Stateless MCP transport)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, In-memory rate limiter with cold-start reset)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, In-memory rate limiter with cold-start reset, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, Structured capture format, prepareThoughtPayload)
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Two-phase pipeline)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Parallel capture pipeline, Retrospective Mode)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Ingestion modes, Parallel capture pipeline)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, Thought prefix format on insert)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Two-phase pipeline)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, upsert_thought RPC)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Self-improvement protocol)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Parallel capture pipeline)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Compaction-safe persistence, In-memory rate limiter with cold-start reset)
- **[recipes/repo-learning-coach/server](../repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType taxonomy)
- **[recipes/repo-learning-coach/src](../repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType taxonomy)
- **[recipes/repo-learning-coach/src/lib](../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType taxonomy)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Hard cost cap with proactive parallelism clamping, In-memory rate limiter with cold-start reset)
- **[recipes/vercel-neon-telegram/src](src/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (In-memory rate limiter with cold-start reset, in-memory sliding-window rate limiter)
- **[recipes/vercel-neon-telegram/src/app/api](src/app/api/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src/lib](src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, captureThought pipeline)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Phase toggles)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Parallel capture pipeline)
- **[server](../../server/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP tool registration, Stateless MCP transport, StreamableHTTPTransport)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Parallel capture pipeline, Retrospective mode)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, In-memory rate limiter with cold-start reset)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Parallel capture pipeline)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType taxonomy, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
