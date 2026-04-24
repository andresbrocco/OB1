# CONTEXT.md — src

## Purpose

Root source directory for the Open Brain SvelteKit dashboard. Contains the application shell, global design tokens, TypeScript ambient declarations, and the server hook that bootstraps Supabase authentication for every request.

## Responsibility Boundaries

- **Owns**: Global CSS theme variables, the HTML document shell, SvelteKit `App` namespace type augmentation, and the server-side auth initialization hook
- **Delegates to**: Route-level layouts and pages (under `src/routes/`) for all page-specific logic; `@supabase/ssr` for cookie-based session management
- **Does not handle**: Individual page rendering, data fetching, or MCP API calls — those live in routes and lib modules

## Key Concepts

- **Thought-type color tokens**: `app.css` defines CSS custom properties for each thought type (`--color-observation`, `--color-task`, `--color-idea`, `--color-reference`, `--color-person-note`). Components across the app rely on these variables rather than hardcoding colors.
- **MCP credential pattern**: `app.d.ts` documents two credential paths for MCP: server-only env vars (`MCP_URL` / `MCP_KEY`) are preferred to avoid leaking keys into the browser bundle; public vars (`PUBLIC_MCP_URL` / `PUBLIC_MCP_KEY`) are a fallback. This distinction is declared at the root type level so it is visible app-wide.

## Non-Obvious Details

- `hooks.server.ts` always sets `event.locals.session = null`. Only `event.locals.user` is populated (from `supabase.auth.getUser()`). Routes that need auth state should check `locals.user`, not `locals.session`.
- The Supabase server client is recreated on every request inside the hook (SvelteKit's recommended SSR pattern); it is not a singleton.
- `PUBLIC_SUPABASE_URL` and `PUBLIC_SUPABASE_ANON_KEY` are required at server startup — missing values throw immediately rather than failing silently at runtime.

## Related Modules

- **[dashboards](../../CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), REST API connection pattern (Next.js dashboard), SvelteKit locals augmentation, iron-session cookie auth)
- **[dashboards/open-brain-dashboard](../CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SSR auth, SvelteKit locals augmentation)
- **[dashboards/open-brain-dashboard-next](../../open-brain-dashboard-next/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SvelteKit locals augmentation, iron-session cookie auth)
- **[dashboards/open-brain-dashboard-next/app/api](../../open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, Thought-type color tokens)
- **[dashboards/open-brain-dashboard-next/components](../../open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Thought-type color tokens)
- **[dashboards/open-brain-dashboard-next/lib](../../open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (SvelteKit locals augmentation, server-only boundary)
- **[dashboards/open-brain-dashboard/src/lib](lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, MCP text-response parsing)
- **[dashboards/open-brain-dashboard/src/routes](routes/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Auth guard via layout.server.ts, SvelteKit locals augmentation, latestResultKey highlight)
- **[docs](../../../docs/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, MCP tool context overhead)
- **[extensions](../../../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Per-request MCP server instantiation)
- **[extensions/family-calendar](../../../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP credential server-vs-public pattern)
- **[extensions/home-maintenance](../../../extensions/home-maintenance/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, SvelteKit locals augmentation)
- **[extensions/household-knowledge](../../../extensions/household-knowledge/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, MCP credential server-vs-public pattern)
- **[extensions/meal-planning](../../../extensions/meal-planning/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Per-request stateless MCP server instantiation)
- **[extensions/professional-crm](../../../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Stateless MCP transport)
- **[integrations](../../../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP credential server-vs-public pattern)
- **[recipes/email-history-import](../../../recipes/email-history-import/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (SvelteKit locals augmentation, Two-layer dedup)
- **[recipes/life-engine](../../../recipes/life-engine/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (SvelteKit locals augmentation, user_id as channel chat_id)
- **[recipes/panning-for-gold](../../../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Thought-type color tokens)
- **[recipes/repo-learning-coach/server](../../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Thought-type color tokens)
- **[recipes/repo-learning-coach/src](../../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Thought-type color tokens)
- **[recipes/repo-learning-coach/src/lib](../../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Thought-type color tokens)
- **[recipes/vercel-neon-telegram](../../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, stateless MCP server)
- **[recipes/vercel-neon-telegram/src/app/api](../../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, MCP protocol compliance (HEAD/DELETE/OPTIONS), Stateless MCP server per request)
- **[recipes/vercel-neon-telegram/src/lib](../../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Thought-type color tokens, ThoughtType)
- **[server](../../../server/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP credential server-vs-public pattern, MCP tool registration, StreamableHTTPTransport)
- **[skills](../../../skills/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SvelteKit locals augmentation, Variants)
- **[skills/heavy-file-ingestion](../../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, SvelteKit locals augmentation)
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Thought-type color tokens, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
