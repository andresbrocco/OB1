# CONTEXT.md — vercel-neon-telegram/src

## Purpose

Full application source for the Vercel + Neon + Telegram recipe. Implements a Next.js App Router server that exposes three intake surfaces (REST API, Telegram bot, MCP protocol) all feeding into a shared AI-powered thought-capture pipeline that embeds and classifies incoming text before writing it to a Neon (Postgres + pgvector) database.

## Responsibility Boundaries

- **Owns**: HTTP route handlers, Telegram webhook logic, MCP tool definitions, auth enforcement, rate limiting, AI calls (embedding + metadata extraction), and database read/write operations for the `thoughts` table
- **Delegates to**: OpenAI (`text-embedding-3-small` for vectors, `gpt-4o-mini` for metadata extraction via Vercel AI SDK), Neon serverless driver for SQL, grammY for Telegram bot lifecycle
- **Does not handle**: Database schema creation or migrations, Telegram webhook registration (must be done externally), OpenAI billing or quota management

## Key Concepts

- **captureThought pipeline** (`lib/capture.ts`): The single shared function used by all three surfaces. It runs embedding generation and metadata extraction in parallel via `Promise.all`, then inserts the result. All sources (api, telegram, mcp) funnel through this function.
- **ThoughtType / ThoughtMetadata** (`lib/types.ts`): Zod-validated schema. Seven thought types: `observation`, `task`, `idea`, `reference`, `person_note`, `decision`, `meeting_note`. Metadata carries `people`, `action_items`, `dates_mentioned`, and up to 3 `topics`.
- **Auth** (`lib/auth.ts`): Single shared secret (`BRAIN_ACCESS_KEY`) checked via `timingSafeEqual` to prevent timing attacks. Accepts the key via `x-brain-key` header, `Authorization: Bearer`, or `?key=` query param. The query param fallback exists explicitly for clients that cannot send custom headers (e.g., ChatGPT plugins).
- **Rate limiter** (`lib/rate-limit.ts`): In-memory sliding window (30 requests/minute). Intentionally resets on cold start — documented as acceptable for personal-use deployments. Its stated purpose is to prevent runaway AI agent loops from burning OpenAI credits.
- **MCP route** (`app/api/mcp/route.ts`): Stateless per-request MCP server using `WebStandardStreamableHTTPServerTransport` with `sessionIdGenerator: undefined`. A new `McpServer` instance is created on every POST. The `HEAD` method returns an `MCP-Protocol-Version` header (`2025-03-26`) required by MCP client discovery. The `DELETE` and `OPTIONS` methods return stubs to satisfy MCP protocol compliance without state.
- **Telegram route** (`app/api/telegram/route.ts`): Module-level singleton bot instance (`let bot`) re-used across warm invocations. Webhook authenticity is verified via `x-telegram-bot-api-secret-token` header matching `TELEGRAM_WEBHOOK_SECRET`. Photo messages are only captured if they include a caption.

## Non-Obvious Details

- The MCP server is stateless by design: each request creates a fresh server and transport, which means there is no session continuity. This is a deliberate tradeoff for Vercel's serverless environment.
- `listThoughts` in `lib/db.ts` builds its WHERE clause dynamically using positional parameters (`$1`, `$2`, …) via `sql.query()`, while `insertThought` and `searchThoughts` use tagged-template SQL. The inconsistency is because the Neon serverless driver's tagged-template form does not support dynamic WHERE clause composition.
- `maxDuration = 15` on the Telegram route and `maxDuration = 30` on the MCP route are Vercel-specific timeout overrides, required because AI API calls may exceed the default 10-second function timeout.
- The Telegram bot guards against double-handling slash commands in the `message:text` handler (`if (ctx.message.text.startsWith("/")) return`) because grammY fires both `bot.command()` and `bot.on("message:text")` for command messages.

## Related Modules

- **[.github](../../../.github/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (PR quota enforcement, in-memory sliding-window rate limiter)
- **[.github/workflows](../../../.github/workflows/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, in-memory sliding-window rate limiter)
- **[dashboards](../../../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier restricted content gating, timingSafeEqual auth)
- **[dashboards/open-brain-dashboard](../../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, timingSafeEqual auth)
- **[dashboards/open-brain-dashboard-next](../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, server-only API proxy, timingSafeEqual auth, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, timingSafeEqual auth)
- **[dashboards/open-brain-dashboard-next/components](../../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, timingSafeEqual auth)
- **[dashboards/open-brain-dashboard-next/lib](../../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, server-only boundary, timingSafeEqual auth, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, stateless MCP server)
- **[dashboards/open-brain-dashboard/src/lib](../../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, stateless MCP server)
- **[dashboards/open-brain-dashboard/src/routes](../../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, timingSafeEqual auth)
- **[docs](../../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, timingSafeEqual auth)
- **[extensions](../../../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, stateless MCP server)
- **[extensions/family-calendar](../../../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, stateless MCP server)
- **[extensions/household-knowledge](../../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, timingSafeEqual auth)
- **[extensions/meal-planning](../../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, timingSafeEqual auth)
- **[extensions/professional-crm](../../../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Stateless MCP transport, stateless MCP server)
- **[integrations](../../../integrations/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, in-memory sliding-window rate limiter)
- **[integrations/entity-extraction-worker](../../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, in-memory sliding-window rate limiter, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, captureThought pipeline, prepareThoughtPayload)
- **[integrations/kubernetes-deployment](../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, timingSafeEqual auth)
- **[recipes/email-history-import](../../email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Ingestion modes, captureThought pipeline)
- **[recipes/google-activity-import](../../google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Thought prefix format on insert, captureThought pipeline)
- **[recipes/instagram-import](../../instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (captureThought pipeline, upsert_thought RPC)
- **[recipes/life-engine](../../life-engine/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Telegram webhook singleton bot, in-memory sliding-window rate limiter, user_id as channel chat_id)
- **[recipes/obsidian-vault-import](../../obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, captureThought pipeline)
- **[recipes/panning-for-gold](../../panning-for-gold/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Compaction-safe persistence, Telegram webhook singleton bot, in-memory sliding-window rate limiter)
- **[recipes/perplexity-conversation-import](../../perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, ThoughtMetadata)
- **[recipes/repo-learning-coach/server](../../repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[recipes/repo-learning-coach/src](../../repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/repo-learning-coach/src/lib](../../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/thought-enrichment](../../thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), timingSafeEqual auth)
- **[recipes/typed-edge-classifier](../../typed-edge-classifier/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Hard cost cap with proactive parallelism clamping, in-memory sliding-window rate limiter)
- **[recipes/vercel-neon-telegram](../CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (In-memory rate limiter with cold-start reset, in-memory sliding-window rate limiter)
- **[recipes/vercel-neon-telegram/src/app/api](app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/lib](lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (ThoughtMetadata)
- **[schemas](../../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier access filtering, timingSafeEqual auth)
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier, timingSafeEqual auth)
- **[server](../../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (timingSafeEqual auth, x-brain-key access key auth)
- **[skills/financial-model-review](../../../skills/financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, ThoughtMetadata)
- **[skills/heavy-file-ingestion](../../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, in-memory sliding-window rate limiter)
- **[skills/n-agentic-harnesses](../../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, ThoughtMetadata)
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
