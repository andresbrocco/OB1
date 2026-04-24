# CONTEXT.md — Open Brain Dashboard (Next)

## Purpose

A full-featured Next.js 14 web dashboard that provides a browser UI for the Open Brain REST API. It handles thought browsing, semantic/text search, smart capture, bulk ingest, kanban workflow management, quality auditing, duplicate resolution, and structured reflections — all behind session-based authentication.

## Responsibility Boundaries

- **Owns**: Session authentication, client-facing UI, Next.js API route proxy layer, kanban status/priority display logic, restricted-content toggle UX
- **Delegates to**: The Open Brain REST API (Supabase Edge Function at `NEXT_PUBLIC_API_URL`) for all data persistence, search, capture, and AI operations
- **Does not handle**: Embedding generation, AI classification, vector search, or any database operations — those are entirely upstream in the REST API

## Key Concepts

- **Session auth with iron-session**: Authentication stores the user's `x-brain-key` API key encrypted in a server-side HTTP-only cookie (`open_brain_session`). The key is never sent to the browser directly. All data fetching happens server-side in RSC pages or through Next.js API route handlers that read the key from the session.
- **Two-layer auth guard**: `middleware.ts` performs a fast cookie-presence check to redirect unauthenticated routes to `/login`. A second server-side check via `requireSession()` / `requireSessionOrRedirect()` validates the session in each API route or server component, providing defense-in-depth.
- **Restricted content gating**: A `restrictedUnlocked` flag on the session controls whether thoughts with a non-default `sensitivity_tier` are included in API responses. The `RestrictedToggle` component is hidden entirely when `RESTRICTED_PASSPHRASE_HASH` is not configured. Unlocking verifies a passphrase against the SHA-256 hash stored in env and sets the session flag; all data views reload on toggle.
- **Kanban workflow**: Only `task` and `idea` thought types participate in the kanban board. Because the REST API supports only single-type filters, `fetchKanbanThoughts` issues two requests (one per type) and merges them client-side, re-sorted by importance descending.
- **Quality audit**: The audit view filters for thoughts with `quality_score` ≤ 29 — considered low-quality by the upstream REST API's scoring — sorted ascending so the worst items appear first.
- **`lib/api.ts` is server-only**: The file is marked `import "server-only"` to prevent accidental use in client components, ensuring the `apiKey` from the session never leaks to the browser bundle.

## Non-Obvious Details

- `NEXT_PUBLIC_API_URL` must be the full Supabase Edge Function URL including the function path (e.g., `.../functions/v1/open-brain-rest`). Despite the `NEXT_PUBLIC_` prefix (which makes it available at build time), it is only consumed server-side in `lib/api.ts`.
- `SESSION_SECRET` is validated at module load time in `lib/auth.ts` with a minimum 32-character length requirement; the app will crash on startup if it is missing or too short.
- `RESTRICTED_PASSPHRASE_HASH` is optional. If omitted, the `RestrictedToggle` component renders nothing and restricted filtering always applies.
- The middleware allows all `/api/*` routes through without a cookie check; individual API route handlers are responsible for their own `requireSession()` call.

## Related Modules

- **[dashboards](../CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, sensitivity_tier restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard](../open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/components](components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/lib](lib/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, restrictedUnlocked, sensitivity_tier, server-only API proxy, server-only boundary, two-layer auth guard, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../open-brain-dashboard/src/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SvelteKit locals augmentation, iron-session cookie auth)
- **[dashboards/open-brain-dashboard/src/lib](../open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard/src/routes](../open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[docs](../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), quality audit (quality_score <= 29))
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, quality audit (quality_score <= 29))
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Source Confidence, quality audit (quality_score <= 29))
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, quality audit (quality_score <= 29))
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, kanban workflow (task/idea types))
- **[recipes/repo-learning-coach/server](../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), kanban workflow (task/idea types))
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, kanban workflow (task/idea types))
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, kanban workflow (task/idea types))
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType taxonomy, kanban workflow (task/idea types))
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, server-only API proxy, timingSafeEqual auth, two-layer auth guard)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, kanban workflow (task/idea types))
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (quality audit (quality_score <= 29), source_confidence)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, sensitivity_tier access filtering, server-only API proxy, two-layer auth guard)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, sensitivity_tier, server-only API proxy, two-layer auth guard)
- **[server](../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard, x-brain-key access key auth)
- **[skills](../../skills/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Variants, iron-session cookie auth)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, quality audit (quality_score <= 29))
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, quality audit (quality_score <= 29))
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, iron-session cookie auth)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (quality audit (quality_score <= 29), quality flags)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT), kanban workflow (task/idea types))
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (quality audit (quality_score <= 29), source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, quality audit (quality_score <= 29))
