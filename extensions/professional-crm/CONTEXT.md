# CONTEXT.md — Professional CRM

## Purpose

Provides an MCP server (Supabase Edge Function) for managing a professional network within Open Brain. Tracks contacts, logs touchpoints, manages deal/opportunity pipelines, and bridges contact records to core Open Brain thoughts.

## Responsibility Boundaries

- **Owns**: `professional_contacts`, `contact_interactions`, and `opportunities` tables; all CRUD and query logic for professional networking data
- **Delegates to**: Core Open Brain `thoughts` table (read-only) for the cross-extension thought-linking tool; Supabase Edge Function runtime for HTTP/MCP transport
- **Does not handle**: Embedding or semantic search on contacts (no pgvector usage); authentication beyond the `MCP_ACCESS_KEY` + `DEFAULT_USER_ID` env-var pattern; scheduling or push-based follow-up reminders

## Key Concepts

- **Interaction log vs. follow-up date**: There are two separate follow-up signals. `contact_interactions.follow_up_needed` + `follow_up_notes` flag that a specific interaction warrants a follow-up. `professional_contacts.follow_up_date` is a top-level date field on the contact used by `get_follow_ups_due` to surface overdue/upcoming tasks — these are not automatically synchronized.
- **`last_contacted` is trigger-managed**: Inserting a row into `contact_interactions` fires a database trigger (`update_contact_last_contacted`) that writes `occurred_at` back to `professional_contacts.last_contacted`. The MCP server code does not update this field directly.
- **Cross-extension bridge**: `link_thought_to_contact` reads from the core `thoughts` table and appends the thought content as a formatted note string into `professional_contacts.notes`. There is no foreign-key relationship — the link is denormalized text.
- **Opportunity stages**: Pipeline stages follow a fixed enum: `identified → in_conversation → proposal → negotiation → won/lost`. Stage transitions are not enforced by the database; the AI client is responsible for advancing them correctly.

## Non-Obvious Details

- The server uses `SUPABASE_SERVICE_ROLE_KEY`, which bypasses RLS. RLS policies are defined in `schema.sql` and apply to direct Supabase client access (e.g., a frontend), but MCP tool calls operate with full service-role access.
- The `get_follow_ups_due` tool returns contacts whose `follow_up_date` is on or before `today + days_ahead`. It always includes overdue contacts (those with a past `follow_up_date`), regardless of the `days_ahead` parameter.
- `link_thought_to_contact` does not verify that `thought.user_id` matches `DEFAULT_USER_ID`. Any thought UUID visible to the service role can be appended to any contact owned by the configured user.
- The MCP transport uses `enableJsonResponse: true` and `sessionIdGenerator: undefined`, making the server stateless. This is intentional for Edge Function deployment — no session state is preserved between calls.
- The Accept-header patching at the top of the POST handler works around connectors that omit `text/event-stream` from their Accept header, which would otherwise cause `@hono/mcp` to reject the request.

## Related Modules

- **[dashboards](../../dashboards/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), Stateless MCP transport)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, Stateless MCP transport)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Stateless MCP transport)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Stateless MCP transport)
- **[docs](../../docs/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP tool context overhead, Stateless MCP transport)
- **[extensions](../CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, Stateless MCP transport)
- **[extensions/family-calendar](../family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Stateless MCP transport)
- **[extensions/household-knowledge](../household-knowledge/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Stateless MCP transport)
- **[extensions/job-hunt](../job-hunt/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Cross-extension bridge via denormalized note append, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Job contact vs professional contact, Pipeline (application status lifecycle), Trigger-managed last_contacted field)
- **[extensions/meal-planning](../meal-planning/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request stateless MCP server instantiation, Stateless MCP transport)
- **[integrations](../../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, Stateless MCP transport)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension bridge via denormalized note append, Dossier thought, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Trigger-managed last_contacted field)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension bridge via denormalized note append, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Pending person confirmation, Trigger-managed last_contacted field)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Stateless MCP transport, stateless MCP server)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request, Stateless MCP transport)
- **[server](../../server/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP tool registration, Stateless MCP transport, StreamableHTTPTransport)
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension bridge via denormalized note append, Diligence packet, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Trigger-managed last_contacted field)
