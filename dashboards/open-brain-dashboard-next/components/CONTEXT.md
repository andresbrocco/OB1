# CONTEXT.md — Components

## Purpose

UI component library for the Open Brain Next.js dashboard. Provides all interactive surface elements for browsing, editing, ingesting, and organizing thoughts stored in the Open Brain Supabase backend.

## Responsibility Boundaries

- **Owns**: All client-side UI rendering and local interaction state (forms, modals, drag-and-drop, optimistic updates)
- **Delegates to**: API routes under `/api/` for all data mutations and reads; `@/lib/types` for shared type definitions and constants
- **Does not handle**: Server-side data fetching (that lives in Next.js page/layout server components), authentication, or direct database access

## Key Concepts

- **KANBAN_TYPES**: A subset of thought types that are eligible to appear on the Kanban board. When a thought's type is changed to one not in `KANBAN_TYPES`, its `status` is nulled and it is removed from the board entirely. This boundary is enforced in both `KanbanCardModal` and `KanbanBoard`.
- **Ingestion modes**: `AddToBrain` supports three ingestion paths — `auto` (backend decides), `single` (one thought), and `extract` (smart-ingest multi-thought extraction). The `extract` path produces an async job. When `showJobDetail` is enabled, the component supports a dry-run preview-then-execute two-phase flow.
- **Reflection types**: `ReflectionComposer` records structured decision traces (`decision_trace`, `lesson_trace`, `retrospective`, `hypothesis`) attached to a specific thought, with weighted factors and enumerated options.
- **Restricted content**: `RestrictedToggle` renders only when the backend has restricted content configured (checked via `/api/restricted` on mount). Unlock sends a passphrase; lock sends a DELETE. Both paths trigger a full page reload to refresh filtered data.

## Non-Obvious Details

- **Optimistic updates with rollback**: `KanbanBoard` stores `previousThoughts` in a `useRef` (not state) before any mutation. On API failure it restores from that ref and shows a transient error banner that auto-clears after 5 seconds. The ref avoids triggering re-renders during the snapshot.
- **Auto-archive logic is client-side**: `KanbanBoard.groupByStatus()` silently moves `done` items older than 30 days (`AUTO_ARCHIVE_DAYS`) into the archived column during grouping. This is a display-only filter — it does not write back to the database unless the user explicitly archives.
- **KanbanCardModal uses React portals**: The modal and its nested delete-confirmation dialog are rendered into `document.body` via `createPortal` to escape stacking context issues. Body scroll is locked on mount and restored on unmount.
- **ThoughtEditor uses a server action prop**: Unlike other components that call fetch directly, `ThoughtEditor` receives an `editAction: (formData: FormData) => Promise<void>` prop (a Next.js Server Action) and invokes it via a `<form action={...}>`. This keeps the mutation on the server side while keeping the toggle-edit UI client-side.
- **TypeBadge is exported from ThoughtCard**: `TypeBadge` is defined in `ThoughtCard.tsx` and re-exported from there. `ConnectionsPanel` imports it directly from that file rather than from a dedicated badge component.
- **ConnectionsPanel renders nothing when no metadata**: The component returns `null` if `hasMetadata` is false or if the connections array is empty after loading, avoiding an empty card in the UI.

## Related Modules

- **[dashboards](../../CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, SSR auth)
- **[dashboards/open-brain-dashboard-next](../CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, Restricted content unlock, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/lib](../lib/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../../open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Thought-type color tokens)
- **[dashboards/open-brain-dashboard/src/lib](../../open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), ThoughtType)
- **[dashboards/open-brain-dashboard/src/routes](../../open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Restricted content passphrase gating)
- **[docs](../../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, Restricted content passphrase gating)
- **[extensions/household-knowledge](../../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Restricted content passphrase gating)
- **[extensions/meal-planning](../../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Restricted content passphrase gating)
- **[integrations](../../../integrations/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Capture integration, Dry-run two-phase ingestion, Ingestion modes (auto/single/extract))
- **[integrations/entity-extraction-worker/_shared](../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), Structured capture format, prepareThoughtPayload)
- **[integrations/kubernetes-deployment](../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Restricted content passphrase gating)
- **[recipes/claudeception](../../../recipes/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Retrospective Mode)
- **[recipes/email-history-import](../../../recipes/email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes, Ingestion modes (auto/single/extract))
- **[recipes/google-activity-import](../../../recipes/google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), Thought prefix format on insert)
- **[recipes/instagram-import](../../../recipes/instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), upsert_thought RPC)
- **[recipes/obsidian-vault-import](../../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Dry-run two-phase ingestion, Ingestion modes (auto/single/extract))
- **[recipes/panning-for-gold](../../../recipes/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[recipes/repo-learning-coach](../../../recipes/repo-learning-coach/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), RepoLearningConfig)
- **[recipes/repo-learning-coach/server](../../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[recipes/repo-learning-coach/src](../../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[recipes/repo-learning-coach/src/lib](../../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[recipes/thought-enrichment](../../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, Sensitivity tiers (standard/personal/restricted))
- **[recipes/vercel-neon-telegram](../../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), Parallel capture pipeline)
- **[recipes/vercel-neon-telegram/src](../../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Restricted content passphrase gating, Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), captureThought pipeline)
- **[schemas](../../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, sensitivity_tier)
- **[server](../../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, x-brain-key access key auth)
- **[skills](../../../skills/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[skills/claudeception](../../../skills/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Retrospective mode)
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
