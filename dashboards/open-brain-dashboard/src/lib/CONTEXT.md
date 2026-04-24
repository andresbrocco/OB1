# CONTEXT.md — lib

## Purpose

Provides the shared data layer for the Open Brain dashboard: type definitions for the `thoughts` domain, a Supabase browser client, and an API client that proxies all data operations through the dashboard's local `/api/mcp` route rather than calling Supabase directly.

## Responsibility Boundaries

- **Owns**: TypeScript type definitions for `Thought` and `ThoughtType`, the Supabase browser client singleton, and all MCP tool invocation and response parsing logic
- **Delegates to**: `/api/mcp` (SvelteKit server route) for actual MCP communication; Supabase for direct database access where needed
- **Does not handle**: UI rendering, routing, server-side auth, or MCP transport configuration

## Key Concepts

- **ThoughtType**: A discriminated union (`observation | task | idea | reference | person_note`) that mirrors the classification schema used by the MCP server's `capture_thought` tool.
- **MCP text-response parsing**: The MCP tools (`list_thoughts`, `search_thoughts`, `thought_stats`) return human-readable plain text, not JSON. `api.ts` contains regex and line-by-line parsers (`parseListResults`, `parseSearchResults`, `parseStatsFromText`) to convert these text payloads into typed `Thought` objects.

## Non-Obvious Details

- MCP responses do not include stable IDs — `api.ts` generates a `crypto.randomUUID()` for each parsed thought. These IDs are ephemeral and will differ across fetches for the same underlying record.
- `list_thoughts` and `search_thoughts` return structurally different text formats; they are handled by separate parsers (`parseListResults` vs `parseSearchResults`). Mixing them up silently produces empty result arrays.
- The Supabase client in `supabase.ts` is a browser client (`createBrowserClient` from `@supabase/ssr`) that requires `PUBLIC_SUPABASE_URL` and `PUBLIC_SUPABASE_ANON_KEY` to be set at runtime; it throws on startup if either is missing.
- `index.ts` is a SvelteKit `$lib` placeholder with no exports; all imports should reference files directly (e.g., `$lib/api`, `$lib/types`).

## Related Modules

- **[dashboards](../../../CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), MCP text-response parsing)
- **[dashboards/open-brain-dashboard](../../CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, MCP text-response parsing)
- **[dashboards/open-brain-dashboard-next](../../../open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../../open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, ThoughtType)
- **[dashboards/open-brain-dashboard-next/components](../../../open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), ThoughtType)
- **[dashboards/open-brain-dashboard-next/lib](../../../open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, ThoughtType)
- **[dashboards/open-brain-dashboard/src](../CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, MCP text-response parsing)
- **[docs](../../../../docs/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, MCP tool context overhead)
- **[extensions](../../../../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Per-request MCP server instantiation)
- **[extensions/family-calendar](../../../../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP text-response parsing)
- **[extensions/home-maintenance](../../../../extensions/home-maintenance/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (ephemeral IDs, frequency_days=NULL for one-time tasks)
- **[extensions/household-knowledge](../../../../extensions/household-knowledge/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, MCP text-response parsing)
- **[extensions/meal-planning](../../../../extensions/meal-planning/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Per-request stateless MCP server instantiation)
- **[extensions/professional-crm](../../../../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Stateless MCP transport)
- **[integrations](../../../../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP text-response parsing)
- **[recipes/fingerprint-dedup-backfill](../../../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Orphan row, ephemeral IDs)
- **[recipes/live-retrieval](../../../../recipes/live-retrieval/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Silent-on-miss contract, ephemeral IDs)
- **[recipes/panning-for-gold](../../../../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, ThoughtType)
- **[recipes/repo-learning-coach/server](../../../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[recipes/repo-learning-coach/src](../../../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/repo-learning-coach/src/lib](../../../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/vercel-neon-telegram](../../../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, stateless MCP server)
- **[recipes/vercel-neon-telegram/src/app/api](../../../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), MCP text-response parsing, Stateless MCP server per request)
- **[recipes/vercel-neon-telegram/src/lib](../../../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType)
- **[schemas/typed-reasoning-edges](../../../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution, ephemeral IDs)
- **[server](../../../../server/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP text-response parsing, MCP tool registration, StreamableHTTPTransport)
- **[skills/panning-for-gold](../../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
