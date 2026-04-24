# CONTEXT.md — lib

## Purpose

Shared server-side infrastructure for the Next.js dashboard. Provides the typed API client for the Open Brain backend, session-based authentication helpers, domain type definitions, and date formatting utilities consumed across server components and API route handlers.

## Responsibility Boundaries

- **Owns**: All HTTP communication with the Open Brain API, iron-session cookie management, domain type contracts, UI-level constants for kanban and priority display
- **Delegates to**: The Open Brain backend API (`NEXT_PUBLIC_API_URL`) for all data storage and retrieval; `iron-session` for cookie encryption
- **Does not handle**: React rendering, route definitions, or direct database access

## Key Concepts

- **`x-brain-key`**: The per-request authentication header forwarded to the backend API. It is stored inside the encrypted session cookie and injected into every `apiFetch` call — it never appears in client-side code.
- **`sensitivity_tier`**: A field on `Thought` that gates access to restricted content. The `exclude_restricted` parameter appears throughout the API layer and defaults to `true` to hide sensitive thoughts unless explicitly unlocked.
- **`restrictedUnlocked`**: A session flag (separate from `loggedIn`) that tracks whether the user has chosen to surface restricted-tier content in the current session.
- **Kanban subset**: Only `task` and `idea` thought types participate in the kanban workflow. This is encoded as `KANBAN_TYPES` in `types.ts` and enforced in `fetchKanbanThoughts`.
- **`AddToBrainMode`**: The `auto` mode lets the backend decide between single-capture and extraction; `single` and `extract` force the path explicitly.

## Non-Obvious Details

- `api.ts` is marked `"server-only"` (Next.js 13+ import guard). Importing it in a client component will throw a build-time error — this is intentional to prevent the API key from leaking to the browser.
- `fetchKanbanThoughts` makes two sequential API calls (one per type) because the backend `/thoughts` endpoint only supports a single `type` filter at a time. Results are re-sorted by importance after merging.
- `auth.ts` exposes two distinct session helpers: `requireSession()` (throws `AuthError` for API route handlers, enabling proper 401 responses) and `requireSessionOrRedirect()` (calls Next.js `redirect()` for server components). Using the wrong one in the wrong context produces either a missed redirect or an unhandled thrown error.
- `SESSION_SECRET` validation runs at module load time. A missing or short secret causes an immediate startup crash rather than a silent misconfiguration.
- `getPriorityLevel` in `types.ts` uses a descending-order `find` — `PRIORITY_LEVELS` must remain sorted from highest `min` to lowest for the lookup to return the correct bucket.

## Related Modules

- **[dashboards](../../CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restrictedUnlocked, sensitivity_tier, sensitivity_tier restricted content gating, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard](../../open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard-next](../CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, restrictedUnlocked, sensitivity_tier, server-only API proxy, server-only boundary, two-layer auth guard, x-brain-key)
- **[dashboards/open-brain-dashboard-next/app/api](../app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard-next/components](../components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../../open-brain-dashboard/src/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (SvelteKit locals augmentation, server-only boundary)
- **[dashboards/open-brain-dashboard/src/lib](../../open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, ThoughtType)
- **[dashboards/open-brain-dashboard/src/routes](../../open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[docs](../../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[extensions](../../../extensions/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pattern, server-only boundary)
- **[extensions/family-calendar](../../../extensions/family-calendar/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pinning, server-only boundary)
- **[extensions/home-maintenance](../../../extensions/home-maintenance/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, server-only boundary)
- **[extensions/household-knowledge](../../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[extensions/meal-planning](../../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[integrations](../../../integrations/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Capture integration)
- **[integrations/entity-extraction-worker/_shared](../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Structured capture format, prepareThoughtPayload)
- **[integrations/kubernetes-deployment](../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[recipes/email-history-import](../../../recipes/email-history-import/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (Two-layer dedup, server-only boundary)
- **[recipes/google-activity-import](../../../recipes/google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Thought prefix format on insert)
- **[recipes/instagram-import](../../../recipes/instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, upsert_thought RPC)
- **[recipes/life-engine](../../../recipes/life-engine/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (server-only boundary, user_id as channel chat_id)
- **[recipes/obsidian-vault-import](../../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Atomic thought)
- **[recipes/panning-for-gold](../../../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, KANBAN_TYPES subset)
- **[recipes/repo-learning-coach/server](../../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), KANBAN_TYPES subset)
- **[recipes/repo-learning-coach/src](../../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, LearningArtifactKind)
- **[recipes/repo-learning-coach/src/lib](../../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, LearningArtifactKind)
- **[recipes/thought-enrichment](../../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[recipes/vercel-neon-telegram](../../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Parallel capture pipeline)
- **[recipes/vercel-neon-telegram/src](../../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, server-only boundary, timingSafeEqual auth, x-brain-key)
- **[recipes/vercel-neon-telegram/src/app/api](../../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[recipes/vercel-neon-telegram/src/lib](../../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, captureThought pipeline)
- **[schemas](../../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, sensitivity_tier access filtering, server-only boundary, x-brain-key)
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[server](../../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key, x-brain-key access key auth)
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
