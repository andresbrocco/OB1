# CONTEXT.md — Dashboards

## Purpose

Frontend templates that provide a browser-based UI for interacting with an Open Brain memory system. Each subdirectory is a self-contained deployable application that connects to a Supabase-hosted backend, allowing users to browse, search, capture, and manage thoughts without using an AI client.

## Responsibility Boundaries

- **Owns**: UI rendering, session/auth handling, client-side routing, and proxying requests to the Open Brain REST API or MCP endpoint
- **Delegates to**: The Open Brain REST API (`open-brain-rest` Supabase Edge Function) or MCP server for all data persistence and retrieval
- **Does not handle**: Embedding generation, vector search execution, or any direct database access — those remain in the backend Edge Functions

## Key Concepts

**Two distinct backend-connection patterns exist across the dashboards:**

- `open-brain-dashboard-next` (Next.js): Connects via the **Open Brain REST API**. Uses iron-session cookie auth where the user's API key is stored server-side in an encrypted cookie. The session secret must be 32+ characters. A `RESTRICTED_PASSPHRASE_HASH` env var (SHA-256 of a passphrase) optionally gates access to thoughts with a `sensitivity_tier` column (requires the `primitives/rls` setup).

- `open-brain-dashboard` (SvelteKit): Connects via the **MCP JSON-RPC protocol** through a server-side proxy route (`/api/mcp`). Uses Supabase Auth (anon key + OAuth) rather than iron-session. MCP responses may be plain JSON or SSE (`data:` lines) — the proxy handles both formats.

**Smart ingest auto-routing** (`open-brain-dashboard-next`): When adding content, an `auto` mode heuristic inspects text length, paragraph count, bullet/numbered lists, speaker-turn patterns, timestamp patterns, and email headers to decide whether to route to single-thought capture or multi-thought extraction.

**Kanban workflow** (`open-brain-dashboard-next`): Only `task` and `idea` thought types participate in the kanban board. Status transitions follow: `new → planning → active → review → done`.

## Non-Obvious Details

- The SvelteKit dashboard parses MCP tool responses by text scraping (regex against formatted plain-text output), not structured JSON payloads from the tool result. This means it is coupled to the specific text formatting of `list_thoughts`, `search_thoughts`, and `thought_stats` MCP tools.
- The Next.js dashboard's middleware (`middleware.ts`) exempts `/api/*` routes from session checks entirely, so API routes must call `requireSession()` themselves — the middleware is not a security boundary for API routes.
- The SvelteKit dashboard supports both `MCP_URL`/`MCP_KEY` (server-only, preferred) and `PUBLIC_MCP_URL`/`PUBLIC_MCP_KEY` (browser-visible, legacy fallback). Using the public variants exposes the MCP key in the browser bundle.
- `_template/` contains only a `metadata.json` stub. It is a scaffold for contributors, not a runnable dashboard.

## Related Modules

- **[dashboards/open-brain-dashboard](open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard-next](open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, sensitivity_tier restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard-next/components](open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard-next/lib](open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restrictedUnlocked, sensitivity_tier, sensitivity_tier restricted content gating, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](open-brain-dashboard/src/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), REST API connection pattern (Next.js dashboard), SvelteKit locals augmentation, iron-session cookie auth)
- **[dashboards/open-brain-dashboard/src/lib](open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), MCP text-response parsing)
- **[dashboards/open-brain-dashboard/src/routes](open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[docs](../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[extensions](../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), Per-request MCP server instantiation)
- **[extensions/family-calendar](../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP JSON-RPC proxy connection pattern (SvelteKit dashboard))
- **[extensions/home-maintenance](../extensions/home-maintenance/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, Supabase Auth)
- **[extensions/household-knowledge](../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[extensions/meal-planning](../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[extensions/professional-crm](../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), Stateless MCP transport)
- **[integrations](../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP JSON-RPC proxy connection pattern (SvelteKit dashboard))
- **[integrations/entity-extraction-worker/_shared](../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, Structured capture format, prepareThoughtPayload)
- **[integrations/kubernetes-deployment](../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[recipes/email-history-import](../recipes/email-history-import/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (Supabase Auth, Two-layer dedup)
- **[recipes/google-activity-import](../recipes/google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, Thought prefix format on insert)
- **[recipes/instagram-import](../recipes/instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, upsert_thought RPC)
- **[recipes/life-engine](../recipes/life-engine/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (Supabase Auth, user_id as channel chat_id)
- **[recipes/obsidian-vault-import](../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Smart ingest auto-routing heuristic)
- **[recipes/panning-for-gold](../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Kanban workflow (task/idea types only))
- **[recipes/repo-learning-coach/server](../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Kanban workflow (task/idea types only))
- **[recipes/repo-learning-coach/src](../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban workflow (task/idea types only), LearningArtifactKind)
- **[recipes/repo-learning-coach/src/lib](../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban workflow (task/idea types only), LearningArtifactKind)
- **[recipes/thought-enrichment](../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), iron-session cookie auth, sensitivity_tier restricted content gating)
- **[recipes/vercel-neon-telegram](../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier restricted content gating, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[recipes/vercel-neon-telegram/src/lib](../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, captureThought pipeline)
- **[schemas](../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier access filtering, sensitivity_tier restricted content gating)
- **[schemas/enhanced-thoughts](../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier, sensitivity_tier restricted content gating)
- **[server](../server/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier restricted content gating, x-brain-key access key auth)
- **[skills](../skills/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), REST API connection pattern (Next.js dashboard), Variants, iron-session cookie auth)
- **[skills/heavy-file-ingestion](../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), REST API connection pattern (Next.js dashboard), iron-session cookie auth)
- **[skills/panning-for-gold](../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban workflow (task/idea types only), Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
