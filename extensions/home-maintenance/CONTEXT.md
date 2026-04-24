# CONTEXT.md — Home Maintenance

## Purpose

An MCP server (Supabase Edge Function) that gives an AI client tools to manage recurring and one-time home maintenance tasks and log completed work. It is the second entry in the curated extensions learning path, introducing a two-table schema with database triggers and a multi-tool MCP surface.

## Responsibility Boundaries

- **Owns**: `maintenance_tasks` and `maintenance_logs` tables, their RLS policies, their triggers, and the four MCP tools that operate on them
- **Delegates to**: Supabase (persistence, RLS enforcement, trigger execution), Claude Desktop (tool invocation via custom connector)
- **Does not handle**: User authentication — identity is pinned to `DEFAULT_USER_ID` env var rather than derived from a JWT or session

## Key Concepts

- **Task vs Log**: `maintenance_tasks` is the authoritative record of what needs doing and when; `maintenance_logs` is the append-only history of completed work. They are separate tables linked by foreign key.
- **Trigger-driven scheduling**: When a log entry is inserted, a PostgreSQL trigger (`update_task_after_log`) automatically updates the parent task's `last_completed` and recalculates `next_due` as `completed_at + frequency_days`. The MCP layer does not perform this calculation itself.
- **frequency_days = NULL**: Signals a one-time task. The trigger sets `next_due` to NULL for such tasks after completion, effectively retiring them from upcoming queries.

## Non-Obvious Details

- **Claude Desktop Accept-header patch**: Claude Desktop connectors do not send the `Accept: text/event-stream` header that `StreamableHTTPTransport` requires. The server detects this and rebuilds the request with the correct header before passing it to the transport. Removing this workaround breaks the connector silently.
- **Access key via query param or header**: Authentication accepts the `MCP_ACCESS_KEY` as either `?key=` query parameter or `x-access-key` header, which supports both URL-embedded and header-based connector configurations in Claude Desktop.
- **Service role key**: The server uses `SUPABASE_SERVICE_ROLE_KEY` (bypasses RLS) rather than an anon key, meaning RLS policies are a safety net for direct DB access, not the enforcement layer for this MCP server.
- **`search_maintenance_history` two-phase query**: History search filters by task attributes (name, category) in a first query to gather task IDs, then filters logs by those IDs. If the first query returns no task IDs the function short-circuits and returns an empty result rather than returning all logs.

## Related Modules

- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, Supabase Auth)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, server-only boundary)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, SvelteKit locals augmentation)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (ephemeral IDs, frequency_days=NULL for one-time tasks)
- **[extensions](../CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, DEFAULT_USER_ID single-tenant pattern)
- **[extensions/family-calendar](../family-calendar/CONTEXT.md)** — Shares Household and Family Data Models domain (NULL family_member_id for household-wide events, maintenance_tasks vs maintenance_logs two-table design, recurring vs. one-time activities in a unified table)
- **[extensions/household-knowledge](../household-knowledge/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID environment injection, DEFAULT_USER_ID identity pinning)
- **[extensions/meal-planning](../meal-planning/CONTEXT.md)** — Shares Household and Family Data Models domain (Dual-server architecture (owner vs household member), Household member RLS via JWT role claim, JSONB ingredient and shopping item storage, Week-anchored meal plan slots, maintenance_tasks vs maintenance_logs two-table design)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Orphan row, frequency_days=NULL for one-time tasks)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, user_id as channel chat_id)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Silent-on-miss contract, frequency_days=NULL for one-time tasks)
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, frequency_days=NULL for one-time tasks, trigger-driven next_due recalculation)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, frequency_days=NULL for one-time tasks, trigger-driven next_due recalculation)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Temporal validity and decay_weight, frequency_days=NULL for one-time tasks, trigger-driven next_due recalculation)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution, frequency_days=NULL for one-time tasks)
