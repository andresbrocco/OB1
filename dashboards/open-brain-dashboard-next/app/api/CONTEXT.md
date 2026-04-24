# CONTEXT.md — app/api

## Purpose

Next.js Route Handler layer that acts as an authenticated proxy and light orchestration layer between the dashboard frontend and the Open Brain backend API. All routes guard against unauthenticated access via `requireSession` and forward requests using the session-stored API key under the `x-brain-key` header.

## Responsibility Boundaries

- **Owns**: Session authentication enforcement, request validation, auto-routing heuristics for ingest, and the restricted-content unlock/lock lifecycle
- **Delegates to**: `@/lib/api` for all actual data operations against the Open Brain backend; `@/lib/auth` for session management
- **Does not handle**: Business logic beyond routing decisions and input validation; embedding, storage, or AI inference (those live in the backend)

## Key Concepts

- **Restricted content**: A second-factor unlock layer on top of session auth. After login, restricted thoughts remain hidden unless the user also submits a passphrase verified against `RESTRICTED_PASSPHRASE_HASH` (SHA-256, env var). Unlock state is stored as `session.restrictedUnlocked` and affects audit, search, kanban, and connections routes via `exclude_restricted`.
- **Auto-routing heuristic (ingest)**: The `POST /api/ingest` endpoint accepts a `mode` of `"auto" | "single" | "extract"`. In `auto` mode it applies `shouldExtract()` — a text-structure heuristic (length, paragraphs, bullets, speaker lines, timestamps, email headers) — to decide whether to call `captureThought` (single thought) or `triggerIngest` (smart multi-thought extraction). `dry_run` is forwarded to the extract path only.
- **Kanban statuses**: Valid values are `new`, `planning`, `active`, `review`, `done`, `archived`. The `archived` status is excluded by default from kanban fetches unless `?archived=true` is passed.
- **Audit quality threshold**: The audit route hard-codes `quality_score_max=29` to surface low-quality thoughts; this threshold is not configurable at runtime.

## Non-Obvious Details

- Destructive operations under `audit/delete` and `kanban/delete` use `POST` rather than `DELETE`, likely for simpler fetch client compatibility.
- `audit/delete` uses `Promise.allSettled` for batch deletes and returns partial success counts (`deleted` + `failed`) rather than failing the whole request if any individual delete errors.
- `duplicates/resolve` with `action: "keep_both"` is a no-op delete — it returns a success response without deleting either thought, which serves as an explicit "dismiss" action.
- Routes that proxy directly to the backend (`thoughts/[id]/connections`, `thoughts/[id]/reflection`, `ingest/[id]`, `ingest/[id]/execute`) construct the URL from `NEXT_PUBLIC_API_URL` directly rather than going through `@/lib/api`, making them bypass any centralized API client logic.
- Auth is always checked before the request body is parsed (`audit/delete` comment explains this explicitly: unauthed requests should get 401, not 400).

## Related Modules

- **[dashboards](../../../CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../../open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, SSR auth, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next](../../CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/components](../../components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, Restricted content unlock, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/lib](../../lib/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../../../open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, Thought-type color tokens)
- **[dashboards/open-brain-dashboard/src/lib](../../../open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, ThoughtType)
- **[dashboards/open-brain-dashboard/src/routes](../../../open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Restricted content unlock, Session-scoped API key forwarding)
- **[docs](../../../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, Restricted content unlock, Session-scoped API key forwarding)
- **[extensions/household-knowledge](../../../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Restricted content unlock, Session-scoped API key forwarding)
- **[extensions/meal-planning](../../../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Restricted content unlock, Session-scoped API key forwarding)
- **[integrations](../../../../integrations/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), Capture integration)
- **[integrations/entity-extraction-worker/_shared](../../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Importance scale (0-6, 6 is user-only))
- **[integrations/kubernetes-deployment](../../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Restricted content unlock, Session-scoped API key forwarding)
- **[recipes/adaptive-capture-classification](../../../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Confidence gating, Per-type thresholds)
- **[recipes/bring-your-own-context](../../../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Source Confidence)
- **[recipes/claudeception](../../../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Quality Gate)
- **[recipes/email-history-import](../../../../recipes/email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), Ingestion modes)
- **[recipes/google-activity-import](../../../../recipes/google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), Thought prefix format on insert)
- **[recipes/instagram-import](../../../../recipes/instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), upsert_thought RPC)
- **[recipes/obsidian-vault-import](../../../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Auto-routing heuristic (shouldExtract))
- **[recipes/panning-for-gold](../../../../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Kanban statuses)
- **[recipes/repo-learning-coach/server](../../../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Kanban statuses)
- **[recipes/repo-learning-coach/src](../../../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, LearningArtifactKind)
- **[recipes/repo-learning-coach/src/lib](../../../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, LearningArtifactKind)
- **[recipes/thought-enrichment](../../../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Sensitivity tiers (standard/personal/restricted), Session-scoped API key forwarding)
- **[recipes/vercel-neon-telegram](../../../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), Parallel capture pipeline)
- **[recipes/vercel-neon-telegram/src](../../../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Restricted content unlock, Session-scoped API key forwarding, Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../../../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), captureThought pipeline)
- **[recipes/work-operating-model-activation](../../../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, source_confidence)
- **[schemas](../../../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, sensitivity_tier)
- **[server](../../../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, x-brain-key access key auth)
- **[skills/claudeception](../../../../skills/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Extraction threshold and quality gates)
- **[skills/financial-model-review](../../../../skills/financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Fatal issues vs. caution flags vs. acceptable simplifications)
- **[skills/heavy-file-ingestion](../../../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Quality flags)
- **[skills/heavy-file-ingestion/scripts](../../../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, quality flags)
- **[skills/panning-for-gold](../../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/work-operating-model](../../../../skills/work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../../../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Five-principle evaluation)
