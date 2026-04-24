# CONTEXT.md — Server

## Purpose

Express HTTP API server for the repo-learning-coach recipe. It synchronizes file-based lesson and research content into Supabase on startup, then serves REST endpoints consumed by the frontend for lesson progress tracking, quiz submission, commenting, and optional Open Brain memory integration.

## Responsibility Boundaries

- **Owns**: HTTP route definitions, request validation, startup orchestration (content sync then listen), Supabase query logic, Open Brain embedding and capture
- **Delegates to**: `content-loader.ts` for parsing markdown files from disk; `repo-learning.config.js` (parent directory) for project-level configuration; Supabase for persistence; OpenRouter API for embeddings
- **Does not handle**: Frontend rendering, markdown file authoring, database schema migrations

## Key Concepts

- **BrainBridge**: The optional integration between the learning coach and the Open Brain `thoughts` table. When `OPENROUTER_API_KEY` is set, the server embeds lesson context and runs a vector similarity search (`match_thoughts`) to surface related prior thoughts for the learner. When the key is absent or the call fails, the bridge degrades gracefully to a disabled state with a human-readable reason rather than throwing.
- **Content sync**: On every server start, `syncContentToSupabase` reads all markdown files from the configured lesson and research directories, upserts them into Supabase, and deletes rows whose source files no longer exist. The sync is additive-safe: existing rows are matched by slug or source path, so renames are handled without duplicate creation.
- **Artifact kinds**: When a learner captures learning output, it is typed as `takeaway`, `confusion`, or `summary`. A `summary` with empty content auto-generates text from lesson metadata; `takeaway` and `confusion` require non-empty content. Each artifact is stored as a `thought` with structured metadata tags.

## Non-Obvious Details

- The server calls `syncContentToSupabase()` before `app.listen()` — the HTTP server does not start until the full sync completes. If the sync fails (e.g., malformed frontmatter or a Supabase error), the process exits rather than serving stale data.
- `SUPABASE_SERVICE_ROLE_KEY` is used (not the anon key), so the server bypasses Row Level Security. This is intentional for a local/self-hosted deployment context but means the server must not be publicly exposed without its own auth layer.
- Quiz question identity in Supabase is keyed on `(quiz_id, order_index)`, not on question text. Reordering questions in the markdown source will update the content of existing rows, not create new ones.
- The `sync-content.ts` file is a standalone CLI entrypoint that calls `syncContentToSupabase` directly, separate from the server startup path. This allows running a one-off sync without starting the HTTP server.

## Related Modules

- **[dashboards](../../../dashboards/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Kanban workflow (task/idea types only))
- **[dashboards/open-brain-dashboard](../../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[dashboards/open-brain-dashboard-next](../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Kanban statuses)
- **[dashboards/open-brain-dashboard-next/components](../../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[dashboards/open-brain-dashboard-next/lib](../../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), KANBAN_TYPES subset)
- **[dashboards/open-brain-dashboard/src](../../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Thought-type color tokens)
- **[dashboards/open-brain-dashboard/src/lib](../../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[docs](../../../docs/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Progressive onboarding sequence)
- **[recipes/claudeception](../../claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Retrospective Mode)
- **[recipes/entity-wiki](../../entity-wiki/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Emergent cached view, UnderstandingState)
- **[recipes/live-retrieval](../../live-retrieval/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Flywheel (read side), UnderstandingState)
- **[recipes/panning-for-gold](../../panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Artifact kinds (takeaway, confusion, summary))
- **[recipes/repo-learning-coach](../CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), Learning Artifacts, LessonStatus, RepoLearningConfig)
- **[recipes/repo-learning-coach/src](../src/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LearningArtifactKind, LessonStatus)
- **[recipes/repo-learning-coach/src/lib](../src/lib/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LearningArtifactKind, LessonStatus)
- **[recipes/vercel-neon-telegram](../../vercel-neon-telegram/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType taxonomy)
- **[recipes/vercel-neon-telegram/src](../../vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[recipes/vercel-neon-telegram/src/lib](../../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[skills](../../../skills/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Lessons Log)
- **[skills/claudeception](../../../skills/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Retrospective mode)
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../../../skills/weekly-signal-diff/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Personalization vs discovery balance, UnderstandingState)
- **[skills/work-operating-model](../../../skills/work-operating-model/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Lean memory discipline, UnderstandingState)
- **[skills/world-model-diagnostic](../../../skills/world-model-diagnostic/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, OB1-connected mode vs. direct-chat mode, UnderstandingState, World-model paradigm (vector database / structured ontology / signal-fidelity))
