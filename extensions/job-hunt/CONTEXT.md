# CONTEXT.md — Job Hunt

## Purpose

A remote MCP server (Supabase Edge Function) that provides a complete job search pipeline manager. It tracks companies, job postings, applications, interviews, and job-specific contacts — all scoped per user via RLS — and exposes each operation as a named MCP tool callable from an AI client.

## Responsibility Boundaries

- **Owns**: The `companies`, `job_postings`, `applications`, `interviews`, and `job_contacts` database tables and all CRUD/query operations on them
- **Delegates to**: Extension 5 (Professional CRM) for persistent relationship management — job contacts can be promoted into `professional_contacts` via `link_contact_to_professional_crm`
- **Does not handle**: Auth token issuance, embedding/semantic search, or general thought capture; access is controlled by a static `MCP_ACCESS_KEY` query param or header rather than user-session tokens

## Key Concepts

- **Pipeline**: The ordered sequence of application statuses (`draft` → `applied` → `screening` → `interviewing` → `offer` → `accepted` / `rejected` / `withdrawn`). `get_pipeline_overview` aggregates counts across all statuses in a single call.
- **Job contact vs professional contact**: `job_contacts` are scoped to a specific job search and may be temporary. Promoting one via `link_contact_to_professional_crm` copies the record into Extension 5's `professional_contacts` table and stores the foreign key in `professional_crm_contact_id`. The FK is application-managed, not enforced at the database level.
- **Interview stages**: Typed as `phone_screen`, `technical`, `behavioral`, `system_design`, `hiring_manager`, `team`, or `final`. Notes captured before the interview (`notes`) and reflection logged after (`feedback` + `rating`) are separate fields on the same row.
- **`learning_order: 6`**: This is the last extension in the curated learning path; it depends on concepts from `deploy-edge-function`, `remote-mcp`, and `rls` primitives.

## Non-Obvious Details

- **Missing `Accept` header workaround**: Claude Desktop connectors omit the `Accept: text/event-stream` header that `StreamableHTTPTransport` requires. The request handler detects this and patches a synthetic header onto a cloned `Request` object before processing. This is a known Claude Desktop connector behavior, not a server bug.
- **`DEFAULT_USER_ID` instead of session auth**: The server uses a single `DEFAULT_USER_ID` environment variable rather than deriving the user from a session token. All Supabase calls use the service role key, meaning RLS policies are bypassed server-side; per-user isolation relies on consistently filtering by `user_id` in every query.
- **Cross-extension write without a formal contract**: `link_contact_to_professional_crm` directly inserts into Extension 5's `professional_contacts` table. There is no shared interface or version check — if Extension 5's schema changes, this tool will silently break.
- **`only_unlinked` filter**: `search_job_contacts` supports `only_unlinked: true` to return contacts where `professional_crm_contact_id IS NULL`, making it easy to find contacts not yet promoted to the CRM.

## Related Modules

- **[extensions/professional-crm](../professional-crm/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Cross-extension bridge via denormalized note append, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Job contact vs professional contact, Pipeline (application status lifecycle), Trigger-managed last_contacted field)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Interview stages, Operating Model Layers)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Dossier thought, Job contact vs professional contact, Pipeline (application status lifecycle))
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Interview stages)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Job contact vs professional contact, Pending person confirmation, Pipeline (application status lifecycle))
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Interview stages, Layer Detail Validators)
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Diligence packet, Job contact vs professional contact, Pipeline (application status lifecycle))
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Interview stages, Structural risk vs. business risk)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Interview stages, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Interview stages, Structural questions framework)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five fixed interview layers, Interview stages)
