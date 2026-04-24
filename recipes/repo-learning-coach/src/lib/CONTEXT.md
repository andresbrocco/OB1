# CONTEXT.md — lib

## Purpose

Shared client-side foundation for the repo-learning-coach frontend: typed API bindings and the canonical TypeScript type definitions used across all UI components.

## Responsibility Boundaries

- **Owns**: All HTTP communication between the frontend and the Next.js API routes; the single source of truth for every shared data shape.
- **Delegates to**: API route handlers (in `src/pages/api/` or `src/app/api/`) for actual data fetching, persistence, and Open Brain integration.
- **Does not handle**: Rendering, state management, routing, or any server-side logic.

## Key Concepts

- **BrainBridgeState**: Represents whether the Open Brain memory layer is available for a given context (`enabled` + a human-readable `reason`). Components check this before offering capture or retrieval features.
- **UnderstandingState**: A learner's self-reported comprehension signal attached to lesson comments — values like `confused`, `want_more_depth`, and `want_examples` drive adaptive follow-up prompts.
- **LearningArtifactKind**: Classifies what a learner captures from a lesson (`takeaway`, `confusion`, `summary`), which determines how the thought is tagged when written to Open Brain.
- **BootstrapData**: A single aggregated payload returned by `/api/bootstrap` that initialises the entire dashboard in one request — includes project metadata, lesson summaries, research documents, and brain bridge state.

## Non-Obvious Details

- `requestJson` is a private generic helper that centralises error extraction: on a non-2xx response it attempts to parse a JSON `{ error }` body before falling back to a generic message. All exported functions delegate to it, so error-handling behaviour is uniform across every API call.
- `RelatedThoughtSummary.similarity` is a cosine-similarity score returned from pgvector; UI code that consumes this should treat it as a float in [0, 1].
- Quiz questions include `explanation` in the detail type but answers (`correctOption`) are also returned server-side after submission — the client never holds the correct answer before submission.

## Related Modules

- **[dashboards](../../../../dashboards/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban workflow (task/idea types only), LearningArtifactKind)
- **[dashboards/open-brain-dashboard](../../../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[dashboards/open-brain-dashboard-next](../../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, LearningArtifactKind)
- **[dashboards/open-brain-dashboard-next/components](../../../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[dashboards/open-brain-dashboard-next/lib](../../../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, LearningArtifactKind)
- **[dashboards/open-brain-dashboard/src](../../../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Thought-type color tokens)
- **[dashboards/open-brain-dashboard/src/lib](../../../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[docs](../../../../docs/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Progressive onboarding sequence)
- **[integrations](../../../../integrations/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Kubernetes self-hosted MCP server)
- **[integrations/kubernetes-deployment](../../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Supabase replacement pattern)
- **[integrations/kubernetes-deployment/k8s](../../../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Co-located pod pattern, ConfigMap-embedded SQL, Supabase schema parity, hostPath volume)
- **[recipes/claudeception](../../../claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Retrospective Mode)
- **[recipes/entity-wiki](../../../entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughtSummary, Semantic expansion)
- **[recipes/live-retrieval](../../../live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), RelatedThoughtSummary, Retrieval log, Session cap (max 3 retrievals))
- **[recipes/local-ollama-embeddings](../../../local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, RelatedThoughtSummary)
- **[recipes/panning-for-gold](../../../panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, LearningArtifactKind)
- **[recipes/repo-learning-coach](../../CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, LearningArtifactKind, RepoLearningConfig)
- **[recipes/repo-learning-coach/server](../../server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LearningArtifactKind, LessonStatus)
- **[recipes/repo-learning-coach/src](../CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, BootstrapData)
- **[recipes/vercel-neon-telegram](../../../vercel-neon-telegram/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType taxonomy)
- **[recipes/vercel-neon-telegram/src](../../../vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/vercel-neon-telegram/src/lib](../../../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[schemas](../../../../schemas/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughtSummary, Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[schemas/enhanced-thoughts](../../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, idempotent schema migration)
- **[server](../../../../server/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughtSummary, match_thoughts RPC (pgvector similarity search))
- **[skills](../../../../skills/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Lessons Log)
- **[skills/claudeception](../../../../skills/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Retrospective mode)
- **[skills/panning-for-gold](../../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../../../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Starter universe bootstrap)
- **[skills/work-operating-model](../../../../skills/work-operating-model/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridgeState, Lean memory discipline, UnderstandingState)
- **[skills/world-model-diagnostic](../../../../skills/world-model-diagnostic/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridgeState, OB1-connected mode vs. direct-chat mode, UnderstandingState, World-model paradigm (vector database / structured ontology / signal-fidelity))
