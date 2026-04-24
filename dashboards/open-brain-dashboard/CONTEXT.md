# CONTEXT.md — Open Brain Dashboard

## Purpose

A production-ready SvelteKit web dashboard for interacting with an Open Brain memory system. Provides UI for searching, filtering, browsing, and capturing thoughts via the Open Brain MCP server, with Supabase-based authentication gating all access.

## Responsibility Boundaries

- **Owns**: UI presentation of thoughts, server-side MCP proxy endpoint, Supabase SSR auth lifecycle, text-parsing of MCP tool responses into typed data structures
- **Delegates to**: The remote MCP Edge Function for all data reads and writes; Supabase for user authentication and session management
- **Does not handle**: Thought storage, embedding generation, or semantic search logic — those live in the MCP Edge Function

## Key Concepts

- **MCP proxy route** (`src/routes/api/mcp/+server.ts`): The browser never calls the MCP server directly. All tool calls go through a SvelteKit server route that injects credentials from private env vars and forwards the JSON-RPC 2.0 request upstream. This keeps `MCP_KEY` out of the browser bundle.
- **Text parsing layer** (`src/lib/api.ts`): MCP tools return human-readable formatted text, not JSON. The dashboard parses these text responses using regex and line-by-line heuristics into typed `Thought` objects. `search_thoughts` and `list_thoughts` return different text formats and are parsed by separate functions (`parseSearchResults` / `parseListResults`).
- **ThoughtType**: The five canonical types (`observation`, `task`, `idea`, `reference`, `person_note`) mirror the MCP server's classification vocabulary and drive UI badge colors.

## Non-Obvious Details

- The MCP proxy handles both plain JSON and SSE (`text/event-stream`) responses from upstream. `parseMcpResponse` detects the format and extracts the last valid JSON-RPC `data:` line if the response is streamed.
- There are two env var strategies for MCP credentials: `MCP_URL`/`MCP_KEY` (server-only, preferred — never exposed to browser) and `PUBLIC_MCP_URL`/`PUBLIC_MCP_KEY` (backward-compatible fallback, visible in browser bundle). The proxy prefers the private vars.
- `Thought` IDs returned from the API layer are generated client-side via `crypto.randomUUID()` because the MCP text responses do not include database row IDs. These IDs are ephemeral and cannot be used for subsequent operations.
- The app is pre-configured for Vercel deployment (`@sveltejs/adapter-vercel`, Node.js 22.x runtime). Svelte 5 runes mode is enabled for all non-`node_modules` files via `dynamicCompileOptions`.
- Auth is enforced at the MCP proxy route: requests without a valid `locals.user` session return 401. The root page layout server also checks session state to redirect unauthenticated users to `/signin`.

## Related Modules

- **[dashboards](../CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard-next](../open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, SSR auth, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/components](../open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, SSR auth)
- **[dashboards/open-brain-dashboard-next/lib](../open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](src/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SSR auth, SvelteKit locals augmentation)
- **[dashboards/open-brain-dashboard/src/lib](src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, MCP text-response parsing)
- **[dashboards/open-brain-dashboard/src/routes](src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, SSR auth)
- **[docs](../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, SSR auth)
- **[extensions](../../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, Per-request MCP server instantiation)
- **[extensions/family-calendar](../../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP proxy route)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, SSR auth)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, SSR auth)
- **[extensions/professional-crm](../../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, Stateless MCP transport)
- **[integrations](../../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP proxy route)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, SSR auth)
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, ThoughtType)
- **[recipes/repo-learning-coach/server](../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, Sensitivity tiers (standard/personal/restricted))
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, SSR auth, Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, sensitivity_tier)
- **[server](../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, x-brain-key access key auth)
- **[skills](../../skills/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SSR auth, Variants)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, SSR auth)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
