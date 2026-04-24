# CONTEXT.md — src

## Purpose

The React frontend for the Repo Learning Coach recipe. It renders a structured onboarding experience backed by Supabase learning tables, letting a user move through ordered lessons, take quizzes, leave understanding-tagged notes, and selectively push durable artifacts into Open Brain's `thoughts` table.

## Responsibility Boundaries

- **Owns**: All UI state for the learning session (active lesson, quiz answers, progress status, confidence score, artifact drafts), API call orchestration, and rendering of lesson content, quiz forms, notes, and research documents.
- **Delegates to**: `/api/*` backend routes (fetched via `src/lib/api.ts`) for all data reads and writes; the backend decides what gets written to `thoughts`.
- **Does not handle**: Quiz grading logic, artifact-to-thought conversion, semantic similarity search for related thoughts — all of that lives in the API layer.

## Key Concepts

- **BrainBridge**: A per-lesson and per-session flag (`{ enabled, reason }`) that gates whether the Open Brain capture panel is shown. When `enabled` is false, `reason` explains why (e.g. the Open Brain connection is not configured). This prevents accidental writes to `thoughts` when the integration is not set up.
- **UnderstandingState**: A learner-declared self-assessment on a note (`clear`, `unsure`, `confused`, `want_more_depth`, `want_examples`). These states tag comments so the backend can prioritize which lessons need revision.
- **LearningArtifactKind**: Three categories of things worth pushing to Open Brain — `takeaway` (a durable insight), `confusion` (an unresolved question to resurface later), and `summary` (auto-generated from lesson state if the body is left blank).
- **Bootstrap**: A single `/api/bootstrap` call loads the full navigation state — project metadata, all lesson summaries with statuses, and the research document list — before any lesson detail is fetched. The `nextRecommendedLesson` field in `dashboard` determines the initial selection.

## Non-Obvious Details

- The app uses `startTransition` when switching lessons or research documents so that React can defer the panel re-render while the new fetch is in flight, keeping the sidebar responsive.
- `refreshLessonAndDashboard` re-fetches both bootstrap and lesson detail in parallel after any mutation (progress save, comment, quiz submit, artifact capture) to keep sidebar stats (quiz best, follow-up count, completion progress) immediately consistent without a full page reload.
- For artifact capture of type `summary`, the body field is optional — the backend generates the summary automatically. The UI skips the minimum-length validation in that case.
- Related thoughts shown in the lesson panel are surfaced by the backend using pgvector similarity against the lesson content; the `similarity` score (0–1) is displayed as a percentage match.

## Related Modules

- **[dashboards](../../../dashboards/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban workflow (task/idea types only), LearningArtifactKind)
- **[dashboards/open-brain-dashboard](../../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[dashboards/open-brain-dashboard-next](../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, LearningArtifactKind)
- **[dashboards/open-brain-dashboard-next/components](../../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[dashboards/open-brain-dashboard-next/lib](../../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, LearningArtifactKind)
- **[dashboards/open-brain-dashboard/src](../../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Thought-type color tokens)
- **[dashboards/open-brain-dashboard/src/lib](../../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[docs](../../../docs/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Progressive onboarding sequence)
- **[integrations](../../../integrations/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Kubernetes self-hosted MCP server)
- **[integrations/kubernetes-deployment](../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Supabase replacement pattern)
- **[integrations/kubernetes-deployment/k8s](../../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Co-located pod pattern, ConfigMap-embedded SQL, Supabase schema parity, hostPath volume)
- **[recipes/claudeception](../../claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Retrospective Mode)
- **[recipes/entity-wiki](../../entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughts, Semantic expansion)
- **[recipes/live-retrieval](../../live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), RelatedThoughts, Retrieval log, Session cap (max 3 retrievals))
- **[recipes/local-ollama-embeddings](../../local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, RelatedThoughts)
- **[recipes/panning-for-gold](../../panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, LearningArtifactKind)
- **[recipes/repo-learning-coach](../CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, LearningArtifactKind, LessonStatus, RepoLearningConfig)
- **[recipes/repo-learning-coach/server](../server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LearningArtifactKind, LessonStatus)
- **[recipes/repo-learning-coach/src/lib](lib/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, BootstrapData)
- **[recipes/vercel-neon-telegram](../../vercel-neon-telegram/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType taxonomy)
- **[recipes/vercel-neon-telegram/src](../../vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/vercel-neon-telegram/src/lib](../../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[schemas](../../../schemas/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughts, Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, idempotent schema migration)
- **[server](../../../server/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughts, match_thoughts RPC (pgvector similarity search))
- **[skills](../../../skills/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Lessons Log)
- **[skills/claudeception](../../../skills/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Retrospective mode)
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Starter universe bootstrap)
- **[skills/work-operating-model](../../../skills/work-operating-model/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Lean memory discipline, UnderstandingState)
- **[skills/world-model-diagnostic](../../../skills/world-model-diagnostic/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, OB1-connected mode vs. direct-chat mode, UnderstandingState, World-model paradigm (vector database / structured ontology / signal-fidelity))
