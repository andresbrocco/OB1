# CONTEXT.md — Household Knowledge

## Purpose

A beginner-level Open Brain extension (learning order: 1) that deploys a remote MCP server as a Supabase Edge Function. It gives an AI client persistent, structured storage for home-management facts: physical items in a home (paint colors, appliances, measurements, documents) and service-provider contacts (vendors). This is the entry point for learning how to build and deploy MCP extensions in the Open Brain ecosystem.

## Responsibility Boundaries

- **Owns**: The `household_items` and `household_vendors` Supabase tables, their RLS policies, and five MCP tool definitions that expose CRUD access to those tables.
- **Delegates to**: Supabase for data persistence, RLS enforcement, and Edge Function hosting; the `deploy-edge-function` and `remote-mcp` primitives for deployment and MCP transport patterns.
- **Does not handle**: Semantic/vector search (uses plain `ilike` text search only), authentication of end users (user identity is injected via the `DEFAULT_USER_ID` environment variable rather than derived from a session token), or any modification of the core `thoughts` table.

## Key Concepts

- **`details` JSONB column**: Household items carry a freeform `details` JSONB field intended for structured metadata (e.g., brand, model number, color code). The schema imposes no shape on this field, so the AI client is responsible for deciding what to store there.
- **`DEFAULT_USER_ID` injection**: Rather than deriving the user from a JWT, the server reads `DEFAULT_USER_ID` from an environment variable at request time. This simplifies single-user deployments but means all requests share a single identity regardless of caller.
- **Accept-header patch**: Claude Desktop does not send the `Accept: text/event-stream` header required by `StreamableHTTPTransport`. The server detects this and rewrites the incoming request before handing it to the MCP transport layer. This is a known workaround, not standard MCP behavior.
- **`MCP_ACCESS_KEY` auth**: All POST requests must supply a pre-shared key via query param (`?key=`) or `x-access-key` header. This is the only caller-authentication mechanism.

## Non-Obvious Details

- The `updated_at` auto-update trigger is defined on `household_items` but not on `household_vendors`, so `household_vendors` has no `updated_at` column at all.
- Search across items uses OR-combined `ilike` filters, not pgvector similarity. There is no embedding or semantic retrieval in this extension.
- The `@ts-ignore` comment on the `duplex: "half"` option in the Accept-header patch is required because Deno's `Request` type does not expose `duplex` in its type definitions, even though it is needed at runtime for streaming bodies.

## Related Modules

- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, SSR auth)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Restricted content unlock, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Restricted content passphrase gating)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, MCP credential server-vs-public pattern)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, MCP text-response parsing)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, MCP_ACCESS_KEY pre-shared key authentication)
- **[docs](../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Query-parameter auth pattern)
- **[extensions](../CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Per-request MCP server instantiation)
- **[extensions/family-calendar](../family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Claude Desktop Accept-header patch)
- **[extensions/home-maintenance](../home-maintenance/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID environment injection, DEFAULT_USER_ID identity pinning)
- **[extensions/meal-planning](../meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, MCP_ACCESS_KEY pre-shared key authentication)
- **[extensions/professional-crm](../professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Stateless MCP transport)
- **[integrations](../../integrations/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Shared config with sensitivity tiers, details JSONB freeform metadata field)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, MCP_ACCESS_KEY pre-shared key authentication)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID environment injection, Two-layer dedup)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID environment injection, user_id as channel chat_id)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, details JSONB freeform metadata field)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Sensitivity tiers (standard/personal/restricted))
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, MCP_ACCESS_KEY pre-shared key authentication, Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (ThoughtMetadata, details JSONB freeform metadata field)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, sensitivity_tier)
- **[server](../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, x-brain-key access key auth)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, details JSONB freeform metadata field)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, details JSONB freeform metadata field)
