# repo-learning-coach

> Interactive learning coach that turns any GitHub repository into a structured course — Express API backend + Vite frontend with curriculum sync, lesson progress tracking, quiz submission, and thought capture.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Supabase project URL | — | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (server-side only) | — | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key for embedding-based brain bridge | — | Yes (if brain bridge enabled) |
| `OPENROUTER_EMBEDDING_MODEL` | Embedding model for related thought retrieval | `openai/text-embedding-3-small` | No |
| `PORT` | Port the Express API listens on | `8787` | No |
| `NODE_ENV` | Runtime environment; set to `production` to serve built frontend | — | No |

Copy `.env.example` to `.env` and fill in values before starting.

### Ports

| Port | Service |
|------|---------|
| `8787` | Express API server (default; override with `PORT`) |
| `5173` | Vite dev server (default Vite port, started by `npm run dev`) |

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/bootstrap` | Returns project metadata, all lessons with progress, research document index, and brain bridge state |
| `GET` | `/api/lessons/:slug` | Full lesson detail including quiz, comments, related research, and brain bridge thoughts |
| `POST` | `/api/lessons/:slug/progress` | Update lesson status and confidence rating |
| `POST` | `/api/lessons/:slug/comments` | Add a lesson comment with understanding state |
| `POST` | `/api/lessons/:slug/capture` | Capture a learning artifact (takeaway, confusion, or summary) to Open Brain |
| `POST` | `/api/quizzes/:quizId/submit` | Submit quiz answers and record attempt score |
| `GET` | `/api/research/:slug` | Full research document detail |

**Example requests:**

```bash
# Load initial app data
curl http://localhost:8787/api/bootstrap

# Get lesson detail
curl http://localhost:8787/api/lessons/intro-to-supabase

# Update lesson progress
curl -X POST http://localhost:8787/api/lessons/intro-to-supabase/progress \
  -H "Content-Type: application/json" \
  -d '{"status": "completed", "confidence": 4}'

# Add a lesson comment
curl -X POST http://localhost:8787/api/lessons/intro-to-supabase/comments \
  -H "Content-Type: application/json" \
  -d '{"body": "This clicked once I saw the RLS example.", "understandingState": "clear"}'

# Capture a takeaway to Open Brain
curl -X POST http://localhost:8787/api/lessons/intro-to-supabase/capture \
  -H "Content-Type: application/json" \
  -d '{"kind": "takeaway", "content": "RLS policies apply before any query reaches your data."}'

# Submit quiz answers
curl -X POST http://localhost:8787/api/quizzes/<quiz-uuid>/submit \
  -H "Content-Type: application/json" \
  -d '{"answers": [{"questionId": "<uuid>", "selectedOption": "B"}]}'

# Get research document detail
curl http://localhost:8787/api/research/rls-deep-dive
```

**Progress status values:** `not_started` | `in_progress` | `completed`

**Confidence range:** `1` (low) – `5` (high)

**Comment understanding states:** `clear` | `unsure` | `confused` | `want_more_depth` | `want_examples`

**Capture kinds:** `takeaway` | `confusion` | `summary`

### Commands

```bash
# Development — starts Express API (tsx watch) and Vite frontend concurrently
npm run dev

# Sync curriculum files to Supabase (run before first start and after editing content)
npm run sync

# Production — compiles TypeScript + Vite, then serves
npm run build
npm run serve

# Lint
npm run lint
```

### Configuration

| File | Purpose |
|------|---------|
| `repo-learning.config.ts` | Project slug, title, curriculum directories, track definition, brain integration settings |
| `.env` / `.env.example` | Runtime environment variables |
| `vite.config.ts` | Vite build and dev server configuration |

`repo-learning.config.ts` is the primary file to edit when deploying this recipe for a new repo. Update `slug`, `title`, `description`, `audience`, `researchDirectory`, and `lessonDirectory` to match your content layout.

### Database Tables

| Table | Purpose |
|-------|---------|
| `repo_learning_projects` | Top-level project record (one per deployed instance) |
| `repo_learning_tracks` | Learning track grouping lessons under a project |
| `repo_learning_lessons` | Individual lesson content, metadata, and goals |
| `repo_learning_lesson_progress` | Per-lesson status, confidence, and quiz score rollups |
| `repo_learning_lesson_comments` | Learner comments with understanding state |
| `repo_learning_research_documents` | Research reference documents synced from the repo |
| `repo_learning_quizzes` | Quiz definition linked to a lesson |
| `repo_learning_quiz_questions` | Individual quiz questions with options and correct answer |
| `repo_learning_quiz_attempts` | Recorded quiz attempt with aggregate score |
| `repo_learning_quiz_responses` | Per-question responses for each attempt |

### Prerequisites

- Node.js 20+
- A Supabase project with the `repo_learning_*` tables provisioned
- OpenRouter API key (only required when brain bridge / related-thought retrieval is enabled)
- Lesson and research Markdown files placed in the directories configured in `repo-learning.config.ts`

## Common Tasks

### First-time setup

```bash
# 1. Install dependencies
npm install

# 2. Copy and fill environment variables
cp .env.example .env
# Edit .env with your SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, OPENROUTER_API_KEY

# 3. Sync curriculum content to Supabase
npm run sync

# 4. Start the development server
npm run dev
# API: http://localhost:8787  |  Frontend: http://localhost:5173
```

### Syncing updated curriculum content

After editing any Markdown files in `researchDirectory` or `lessonDirectory`:

```bash
npm run sync
```

The sync is also run automatically on server start (`npm run serve`), so in production a redeploy is sufficient.

### Deploying to production

```bash
npm run build        # TypeScript compile + Vite bundle
npm run serve        # Express serves API + static frontend from dist/
```

In production the Express server serves the built frontend as static files, so only port `8787` needs to be exposed.

### Adapting for a new repository

1. Edit `repo-learning.config.ts` — update `slug`, `title`, `description`, `audience`, and point `researchDirectory`/`lessonDirectory` to your content folders.
2. Place research Markdown files in the configured research directory.
3. Place lesson Markdown files (with quiz frontmatter) in the configured lesson directory.
4. Run `npm run sync` to push content to Supabase.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Project <slug> was not found. Run npm run sync first.` | Curriculum has never been synced | Run `npm run sync` before starting the server |
| `Lesson <slug> was not found.` on GET `/api/lessons/:slug` | Slug does not match any synced lesson | Confirm the slug in the URL matches the Markdown filename; re-run `npm run sync` |
| Brain bridge returns `enabled: false` | `OPENROUTER_API_KEY` is missing or embedding call failed | Set `OPENROUTER_API_KEY` in `.env`; check OpenRouter quota |
| `Invalid lesson progress payload.` (400) | `status` or `confidence` outside allowed values | `status` must be one of `not_started` / `in_progress` / `completed`; `confidence` must be an integer 1–5 |
| Vite proxy errors in dev | Express not yet listening when Vite starts | Wait a moment; `concurrently` starts both processes simultaneously — the API may need a second to boot |
| `Failed to sync research document <slug>.` during sync | Supabase write error or missing table | Confirm the `repo_learning_*` tables exist in your Supabase project |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [server/CONTEXT.md](server/CONTEXT.md) — Server-side implementation details
- [src/CONTEXT.md](src/CONTEXT.md) — Frontend implementation details
- [src/lib/CONTEXT.md](src/lib/CONTEXT.md) — Frontend API client and types
