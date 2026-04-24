# CONTEXT.md — Api

## Purpose

Defines all HTTP route handlers for the Vercel-Neon-Telegram recipe. Each subdirectory is a Next.js App Router API route that exposes a distinct ingestion or access surface: REST capture, MCP protocol, Telegram bot webhook, and health check.

## Responsibility Boundaries

- **Owns**: HTTP request validation, authentication enforcement, rate limiting at the edge, and delegating to shared library functions
- **Delegates to**: `@/lib/capture` (thought classification and persistence), `@/lib/db` (search and list queries), `@/lib/ai` (embedding generation), `@/lib/auth` (bearer token validation), `@/lib/rate-limit` (in-memory rate limiting)
- **Does not handle**: Database access directly, embedding generation, or AI classification — all of that lives in `lib/`

## Key Concepts

- **MCP route (`/api/mcp`)**: Implements the Model Context Protocol over HTTP using `WebStandardStreamableHTTPServerTransport`. A new `McpServer` instance is created per request (stateless), meaning no persistent session state is held across calls. The `HEAD` method returns `MCP-Protocol-Version` and `DELETE`/`OPTIONS` are no-ops required for protocol compliance.
- **Telegram route (`/api/telegram`)**: Receives Telegram webhook payloads from the Bot API. Authentication is via `x-telegram-bot-api-secret-token` header (set when registering the webhook with Telegram), not the bearer token used by other routes. The `Bot` instance is a module-level singleton to avoid re-initialization on warm Lambda invocations.
- **Capture route (`/api/capture`)**: Simple REST endpoint for programmatic thought ingestion. Uses the same bearer token auth and rate limiter as the MCP route. Source is recorded as `"api"`.

## Non-Obvious Details

- The MCP and capture routes share `requireAuth` (bearer token) and `checkRateLimit`, but the Telegram route uses its own secret-token check — mixing these up would break webhook delivery since Telegram does not send a bearer token.
- `maxDuration` is set at the top of `mcp/route.ts` (30s) and `telegram/route.ts` (15s) as Vercel function timeout hints; the health route has no limit because it returns immediately.
- The bot singleton (`let bot: Bot | undefined`) is intentional: Vercel serverless functions can be reused across requests on the same warm instance, so re-creating the bot each time would add unnecessary overhead. However, this means `TELEGRAM_BOT_TOKEN` is read once at first invocation — a token rotation requires a cold start.
- The `/api/mcp` GET handler returns a 405 with an explanatory message rather than a true 405 status, to inform MCP clients that the endpoint is stateless and POST-only for all operations.

## Related Modules

- **[dashboards](../../../../../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../../../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, SSR auth, Telegram webhook secret authentication)
- **[dashboards/open-brain-dashboard-next](../../../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Restricted content unlock, Session-scoped API key forwarding, Telegram webhook secret authentication)
- **[dashboards/open-brain-dashboard-next/components](../../../../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Restricted content passphrase gating, Telegram webhook secret authentication)
- **[dashboards/open-brain-dashboard-next/lib](../../../../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../../../../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request)
- **[dashboards/open-brain-dashboard/src/lib](../../../../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), MCP text-response parsing, Stateless MCP server per request)
- **[dashboards/open-brain-dashboard/src/routes](../../../../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Bearer token authentication, Telegram webhook secret authentication)
- **[docs](../../../../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Query-parameter auth pattern, Telegram webhook secret authentication)
- **[extensions](../../../../../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), Per-request MCP server instantiation, Stateless MCP server per request)
- **[extensions/family-calendar](../../../../../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request)
- **[extensions/household-knowledge](../../../../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, MCP_ACCESS_KEY pre-shared key authentication, Telegram webhook secret authentication)
- **[extensions/meal-planning](../../../../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Household member RLS via JWT role claim, Telegram webhook secret authentication)
- **[extensions/professional-crm](../../../../../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request, Stateless MCP transport)
- **[integrations](../../../../../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request)
- **[integrations/kubernetes-deployment](../../../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, MCP_ACCESS_KEY authentication, Telegram webhook secret authentication)
- **[recipes/life-engine](../../../../life-engine/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Bot singleton pattern, Telegram webhook secret authentication, user_id as channel chat_id)
- **[recipes/panning-for-gold](../../../../panning-for-gold/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Bot singleton pattern, Compaction-safe persistence, Telegram webhook secret authentication)
- **[recipes/thought-enrichment](../../../../thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Sensitivity tiers (standard/personal/restricted), Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram](../../../CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, timingSafeEqual auth)
- **[schemas](../../../../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, sensitivity_tier)
- **[server](../../../../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, x-brain-key access key auth)
