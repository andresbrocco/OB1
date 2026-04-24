# CONTEXT.md — Meal Planning

## Purpose

A Supabase-backed MCP extension for recipe management, weekly meal planning, and household-shared grocery lists. It is the fourth step in the curated extension learning path and introduces the `shared-mcp` primitive — the first extension that demonstrates multi-user, scoped household access on top of Open Brain.

## Responsibility Boundaries

- **Owns**: `recipes`, `meal_plans`, and `shopping_lists` tables; their RLS policies; and both the owner and household-member MCP servers.
- **Delegates to**: The `deploy-edge-function`, `remote-mcp`, `rls`, and `shared-mcp` primitives for deployment and security patterns.
- **Does not handle**: Semantic search over recipes (no pgvector usage), push notifications, or calendar integration.

## Key Concepts

- **Dual-server architecture**: `index.ts` is the owner's full-access MCP server (all CRUD tools). `shared-server.ts` is a separate, limited server for household members — read-only on recipes and meal plans, update-only on shopping lists. Each server uses a distinct Supabase key and a distinct `MCP_ACCESS_KEY`/`MCP_HOUSEHOLD_ACCESS_KEY` secret.
- **Household member role**: RLS SELECT policies on all three tables permit rows where `auth.jwt() ->> 'role' = 'household_member'`. The household server is expected to connect with a Supabase key that carries this JWT claim. Shopping lists additionally grant UPDATE to household members so they can mark items purchased.
- **JSONB for structured sub-objects**: `ingredients` (array of `{name, quantity, unit}`) and `instructions` (array of strings) are stored as JSONB columns, not child tables. The `shopping_lists.items` column follows the same pattern and includes a `purchased` boolean flag per item.
- **Week-anchored meal plans**: Meals are keyed by `week_start` (expected to be a Monday in `YYYY-MM-DD` form). Each row is one meal slot (`day_of_week` + `meal_type`). There is no unique constraint enforcing one entry per slot, so duplicate insertions are possible.

## Non-Obvious Details

- **Ingredient quantity aggregation is naive**: `generate_shopping_list` concatenates duplicate ingredient quantities as a string (`"1 + 2"`) rather than summing them numerically. A comment in the code acknowledges this as a known limitation.
- **Claude Desktop `Accept` header patch**: `index.ts` detects when the incoming request lacks `text/event-stream` in the `Accept` header (a known Claude Desktop connector quirk) and rebuilds the request with the correct value before passing it to `StreamableHTTPTransport`. The `shared-server.ts` omits this workaround.
- **Per-request server instantiation**: Both servers construct a new `McpServer` instance on every HTTP POST. This is stateless by design for Edge Function compatibility, but means there is no persistent in-process state between calls.
- **`DEFAULT_USER_ID` env var**: The owner server reads the target user from this environment variable rather than from the authenticated JWT. This is intentional for single-owner deployments but means the server does not enforce per-request authentication at the user level beyond the access key check.

## Related Modules

- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, SSR auth)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Restricted content unlock, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Restricted content passphrase gating)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Per-request stateless MCP server instantiation)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Per-request stateless MCP server instantiation)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Household member RLS via JWT role claim)
- **[docs](../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Query-parameter auth pattern)
- **[extensions](../CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, Per-request stateless MCP server instantiation)
- **[extensions/family-calendar](../family-calendar/CONTEXT.md)** — Shares Household and Family Data Models domain (Dual-server architecture (owner vs household member), Household member RLS via JWT role claim, JSONB ingredient and shopping item storage, NULL family_member_id for household-wide events, Week-anchored meal plan slots, recurring vs. one-time activities in a unified table)
- **[extensions/home-maintenance](../home-maintenance/CONTEXT.md)** — Shares Household and Family Data Models domain (Dual-server architecture (owner vs household member), Household member RLS via JWT role claim, JSONB ingredient and shopping item storage, Week-anchored meal plan slots, maintenance_tasks vs maintenance_logs two-table design)
- **[extensions/household-knowledge](../household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, MCP_ACCESS_KEY pre-shared key authentication)
- **[extensions/professional-crm](../professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request stateless MCP server instantiation, Stateless MCP transport)
- **[integrations](../../integrations/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, Shared config with sensitivity tiers)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, MCP_ACCESS_KEY authentication)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, JSONB ingredient and shopping item storage)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Sensitivity tiers (standard/personal/restricted))
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request stateless MCP server instantiation, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Household member RLS via JWT role claim, Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, ThoughtMetadata)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, sensitivity_tier)
- **[server](../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, x-brain-key access key auth)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, Model shape)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, Product shape)
