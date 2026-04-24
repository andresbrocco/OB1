# CONTEXT.md — Repo Learning Coach

## Purpose

A locally-run learning application that turns file-based curriculum content into a Supabase-backed interactive workspace — with lesson progress tracking, quizzes, per-lesson comments, and the ability to capture learning artifacts (takeaways, confusions, summaries) durably into Open Brain's `thoughts` table.

## Responsibility Boundaries

- **Owns**: Parsing and syncing markdown curriculum files into Supabase on startup; serving a REST API for lesson/quiz/progress/capture operations; bridging lesson context to Open Brain via semantic search and thought upserts.
- **Delegates to**: Supabase (persistence of all learning state); OpenRouter (embeddings for brain bridge); the React frontend (`src/`) for all UI concerns.
- **Does not handle**: Authentication, multi-user isolation, or curriculum authoring — all content is file-based and single-user by design.

## Key Concepts

- **Brain Bridge**: The optional integration layer (`server/brain.ts`) that connects lessons to Open Brain. When `OPENROUTER_API_KEY` is set, it embeds lesson context and calls `match_thoughts` to surface related prior thoughts, and `upsert_thought` to write captures back. When the key is absent, the bridge degrades gracefully (returns empty results, reports its own disabled state via `BrainBridgeState`).
- **Content-Runtime Separation**: Curriculum lives as markdown files in `research/` and `curriculum/lessons/`. On every server start, `syncContentToSupabase` reads all markdown via `content-loader.ts`, diffs against existing Supabase rows by slug and source path, upserts changed records, and hard-deletes stale ones. The app at runtime reads only from Supabase, never from disk directly.
- **RepoLearningConfig**: A single typed config object in `repo-learning.config.ts` is the single source of truth for project slug, track metadata, directory paths, and brain integration settings. This is the main customization point when adapting the recipe for a new repo.
- **Learning Artifacts**: Captures from a lesson come in three kinds — `takeaway` (explicit insight), `confusion` (follow-up question), and `summary` (auto-generated from lesson metadata when no content is provided). Each is formatted into a natural-language string before embedding and upsert.
- **Understanding State**: Comments on lessons carry a typed `understanding_state` (`clear`, `unsure`, `confused`, `want_more_depth`, `want_examples`). States of `confused`, `want_more_depth`, and `want_examples` are counted as "follow-ups" and surfaced in the dashboard.

## Non-Obvious Details

- `syncContentToSupabase` runs on every server start (not as a separate CLI command), so the Supabase tables are always in sync with the filesystem before the first request is served. Running `npm run sync` also calls this directly without starting the HTTP server.
- Slug identity for research documents uses both slug and `source_path` as lookup keys during sync. This means renaming a file without updating its frontmatter `slug` field will be treated as a rename (matched by slug), while changing the slug without changing the file path is also handled (matched by path). Only if both change will a new row be inserted and the old one deleted.
- Quiz question correctness validation happens at load time in `content-loader.ts` — if `correctOption` is not present in the `options` array, the entire server startup fails. Similarly, `relatedResearch` slugs are validated against the loaded research set, so referencing a nonexistent research slug is a hard error at startup.
- The brain bridge captures are two-step: `upsert_thought` (via RPC) creates the row and returns its id, then a separate `.update({ embedding })` call patches the embedding. If the embedding call fails, the thought still exists in the database but without a vector.
- All Supabase queries use the service role key, so RLS is bypassed entirely. This recipe is not designed for public or authenticated user access.

## Related Modules

- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), RepoLearningConfig)
- **[docs](../../docs/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, Progressive onboarding sequence, RepoLearningConfig)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, RepoLearningConfig, Retrospective Mode)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Emergent cached view, Understanding State)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Flywheel (read side), Understanding State)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Flywheel closure, Understanding State)
- **[recipes/repo-learning-coach/server](server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), Learning Artifacts, LessonStatus, RepoLearningConfig)
- **[recipes/repo-learning-coach/src](src/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, LearningArtifactKind, LessonStatus, RepoLearningConfig)
- **[recipes/repo-learning-coach/src/lib](src/lib/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, LearningArtifactKind, RepoLearningConfig)
- **[skills](../../skills/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, Lessons Log, RepoLearningConfig)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, RepoLearningConfig, Retrospective mode)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Personalization vs discovery balance, Understanding State)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Lean memory discipline, Understanding State)
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, OB1-connected mode vs. direct-chat mode, Understanding State, World-model paradigm (vector database / structured ontology / signal-fidelity))
