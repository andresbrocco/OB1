# CONTEXT.md — Family Calendar

## Purpose

A remote MCP server (Supabase Edge Function) that provides household scheduling capabilities to AI clients. It manages a roster of family members, their activities (both one-time and recurring), and important dates such as birthdays and anniversaries.

## Responsibility Boundaries

- **Owns**: Family member records, activity scheduling, important date tracking, and week-view query assembly
- **Delegates to**: Supabase for persistence and query execution; `@hono/mcp` / `StreamableHTTPTransport` for MCP protocol handling
- **Does not handle**: Push notifications or reminder delivery (reminder_days_before is stored but no dispatch logic exists), authentication beyond a single shared `MCP_ACCESS_KEY`, or multi-user tenancy beyond `DEFAULT_USER_ID`

## Key Concepts

- **Recurring vs. one-time activities**: Both live in the same `activities` table. One-time events have `day_of_week = NULL` and a `start_date`. Recurring events carry a `day_of_week` string (e.g. `'monday'`) and optionally `start_date`/`end_date` bounds. The `get_week_schedule` tool combines both in a single `.or()` filter.
- **family_member_id = NULL**: In both `activities` and `important_dates`, a NULL `family_member_id` means the event belongs to the whole household rather than any individual member.
- **learning_order: 3**: This is the third step in the curated Open Brain extension learning path, positioned after two prerequisite extensions and before more advanced ones.

## Non-Obvious Details

- **Claude Desktop `Accept` header patch**: Claude Desktop connectors do not send the `Accept: text/event-stream` header that `StreamableHTTPTransport` requires. The server detects this and rebuilds the request with the correct header before handing it off to the transport. The `duplex: "half"` property on the patched `Request` is required by Deno for streaming request bodies and is suppressed from TypeScript's type checker via `@ts-ignore` because the `RequestInit` type does not include it.
- **`DEFAULT_USER_ID` environment variable**: Rather than deriving the user from a session token, the server pins all writes and reads to a single user ID set at deploy time. This is by design for single-household deployments but means the Edge Function is not multi-tenant.
- **`get_week_schedule` query semantics**: The `.or()` filter matches activities that either (a) fall within the requested date range as one-time events, or (b) have a non-null `day_of_week` as recurring events. Filtering by date bounds for recurring events still relies on `start_date`/`end_date` guards in the same clause, so recurring events without a `start_date` will always appear regardless of the requested week.
- **No `DEFAULT_USER_ID` in `.env.example`**: The `.env.example` lists only the three Supabase/MCP variables. `DEFAULT_USER_ID` must be set separately as a Supabase Edge Function secret.

## Related Modules

- **[dashboards](../../dashboards/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP JSON-RPC proxy connection pattern (SvelteKit dashboard))
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP proxy route)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pinning, server-only boundary)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP credential server-vs-public pattern)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP text-response parsing)
- **[docs](../../docs/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP tool context overhead)
- **[extensions](../CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Per-request MCP server instantiation)
- **[extensions/home-maintenance](../home-maintenance/CONTEXT.md)** — Shares Household and Family Data Models domain (NULL family_member_id for household-wide events, maintenance_tasks vs maintenance_logs two-table design, recurring vs. one-time activities in a unified table)
- **[extensions/household-knowledge](../household-knowledge/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Claude Desktop Accept-header patch)
- **[extensions/meal-planning](../meal-planning/CONTEXT.md)** — Shares Household and Family Data Models domain (Dual-server architecture (owner vs household member), Household member RLS via JWT role claim, JSONB ingredient and shopping item storage, NULL family_member_id for household-wide events, Week-anchored meal plan slots, recurring vs. one-time activities in a unified table)
- **[extensions/professional-crm](../professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Stateless MCP transport)
- **[integrations](../../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Kubernetes self-hosted MCP server)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pinning, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (NULL family_member_id for household-wide events, Orphan row)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pinning, user_id as channel chat_id)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (NULL family_member_id for household-wide events, Silent-on-miss contract)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, stateless MCP server)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request)
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, recurring vs. one-time activities in a unified table)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, recurring vs. one-time activities in a unified table)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Temporal validity and decay_weight, recurring vs. one-time activities in a unified table)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (NULL family_member_id for household-wide events, Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution)
- **[server](../../server/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, Claude Desktop Accept-header patch, MCP tool registration, StreamableHTTPTransport)
