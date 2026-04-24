# DEPENDENCIES.md

> Dependency graph for the Open Brain (OB1) codebase.

## Internal Module Dependencies

How modules within this codebase depend on each other.

### Core Modules

| Module | Purpose | Depended On By |
|--------|---------|----------------|
| `server/` | Core MCP server: thought capture, semantic search, memory retrieval via Supabase Edge Function | dashboards (remote MCP calls), recipes/live-retrieval, recipes/repo-learning-coach, recipes/claudeception, skills/claudeception, skills/panning-for-gold, skills/work-operating-model |
| `schemas/entity-extraction` | Knowledge graph schema: `entities`, `edges`, `thought_entities`, async extraction queue | integrations/entity-extraction-worker, schemas/typed-reasoning-edges, recipes/wiki-compiler |
| `schemas/enhanced-thoughts` | Extends `thoughts` with classification columns (`thought_type`, `quality_score`, `sensitivity_tier`, etc.) | integrations/entity-extraction-worker, recipes/thought-enrichment, recipes/adaptive-capture-classification |
| `schemas/typed-reasoning-edges` | Adds typed semantic `thought_edges` table (requires entity-extraction applied first) | recipes/typed-edge-classifier |
| `.github/` | Contribution governance: CI gate rules, metadata schema, PR template | .github/workflows/ |
| `.github/workflows/` | GitHub Actions automation for the full contribution lifecycle | (executed by GitHub Actions runner) |
| `docs/` | End-user setup guides and companion prompts | .github/ (gate failure messages reference docs/) |
| `integrations/entity-extraction-worker/_shared/` | Shared entity extraction pipeline utilities (config, LLM helpers) | integrations/entity-extraction-worker/ |
| `recipes/repo-learning-coach/server/` | Express HTTP API and Supabase sync for the learning coach | recipes/repo-learning-coach/src/ |
| `recipes/repo-learning-coach/src/lib/` | Shared typed API client and TypeScript types | recipes/repo-learning-coach/src/ |
| `recipes/vercel-neon-telegram/src/lib/` | Core shared library: auth, DB, embeddings, capture, rate-limit | recipes/vercel-neon-telegram/src/app/api/ |
| `dashboards/open-brain-dashboard-next/lib/` | Typed API client, session management, Supabase admin client | dashboards/open-brain-dashboard-next/app/api/, dashboards/open-brain-dashboard-next/components/ |

### Module Dependency Graph

```
[.github/workflows/] ──────────> [.github/] (metadata.schema.json, templates)
                    └──────────> [anthropics/claude-code-action@v1] (external)
                    └──────────> [release-drafter/release-drafter@v6] (external)

[server/] ─────────────────────> [Supabase] (upsert_thought RPC, match_thoughts RPC)
                    └──────────> [OpenRouter API] (embeddings + LLM metadata extraction)

[dashboards/open-brain-dashboard-next/]
  app/api/ ──────────────────> lib/ (api.ts, auth.ts)
  components/ ───────────────> (React, Tailwind CSS — no internal cross-module imports)
  (dashboard as a whole) ────> [server/] (remote REST/MCP call via NEXT_PUBLIC_API_URL)

[dashboards/open-brain-dashboard/]
  src/routes/ ───────────────> src/lib/ (Supabase client, vector search types)
  src/routes/api/mcp/ ───────> [server/] (remote MCP call — credentials injected server-side)

[integrations/entity-extraction-worker/]
  index.ts ──────────────────> _shared/ (config.ts, helpers.ts)
  (worker as a whole) ───────> [schemas/entity-extraction] (reads queue, writes entities/edges)
  (worker as a whole) ───────> [schemas/enhanced-thoughts] (reads metadata column)
  (worker as a whole) ───────> [Anthropic Claude API / OpenRouter] (LLM extraction)

[schemas/typed-reasoning-edges] ──> [schemas/entity-extraction] (prerequisite: extends edges table)

[recipes/repo-learning-coach/]
  src/ ──────────────────────> server/ (HTTP REST calls)
  src/lib/ ──────────────────> (shared types consumed by src/ components)
  server/ ───────────────────> [server/] (brain bridge: match_thoughts, upsert_thought RPCs)

[recipes/vercel-neon-telegram/]
  src/app/api/ ──────────────> src/lib/ (auth, db, ai, capture, rate-limit)
  src/lib/ ──────────────────> [Neon Postgres + OpenAI API] (external)

[recipes/wiki-compiler/] ──────> [schemas/entity-extraction] (reads entities table)
                         └────> [recipes/wiki-synthesis/] (sequences its pipeline)

[recipes/typed-edge-classifier/] ──> [schemas/typed-reasoning-edges] (writes thought_edges)

[skills/panning-for-gold/] ────> [server/] (upsert_thought — remote MCP call)
[skills/work-operating-model/] ─> [server/] (capture_thought — remote MCP call)
[skills/claudeception/] ────────> [server/] (upsert_thought — remote MCP call)
[recipes/claudeception/] ───────> [server/] (upsert_thought — remote MCP call)
[recipes/life-engine/] ─────────> [server/] (search_thoughts, capture_thought — remote MCP call)
[recipes/live-retrieval/] ──────> [server/] (search_thoughts MCP tool — remote MCP call)
```

### Dependency Details

#### `integrations/entity-extraction-worker/index.ts` → `integrations/entity-extraction-worker/_shared/`
- Imports `helpers.ts`: LLM call wrappers with AbortController timeout, entity/edge upsert helpers, prompt injection defense (XML delimiter escaping)
- Imports `config.ts`: importance scale constants (0–6), sensitivity tier regex patterns, allowed thought type allowlist, LLM provider fallback order (OpenRouter → OpenAI → Anthropic)

#### `integrations/entity-extraction-worker/` → `schemas/entity-extraction`
- Reads and drains `entity_extraction_queue` with atomic status transitions (`pending` → `processing` → `done`/`skipped`)
- Writes extracted entities to `entities`, relationship evidence to `edges` (with `support_count` increment), and thought-entity links to `thought_entities`
- Cleans up stale `thought_entities` links (`source='entity_worker'`) before re-writing on content change

#### `integrations/entity-extraction-worker/` → `schemas/enhanced-thoughts`
- Reads `thoughts.metadata->>'generated_by'` to detect and skip system-generated thought rows

#### `schemas/typed-reasoning-edges` → `schemas/entity-extraction`
- Hard apply-order prerequisite: migration adds `valid_from`, `valid_until`, and `decay_weight` columns to the `edges` table introduced by `entity-extraction`
- Applying out of order raises a runtime exception with an actionable message

#### `dashboards/open-brain-dashboard-next/app/api/` → `dashboards/open-brain-dashboard-next/lib/`
- All API route handlers import `requireSession`, `getSession`, `AuthError` from `lib/auth`
- Data-fetching routes import typed functions (`fetchThoughts`, `searchThoughts`, `deleteThought`, `fetchDuplicates`) from `lib/api`

#### `dashboards/open-brain-dashboard/src/routes/` → `dashboards/open-brain-dashboard/src/lib/`
- Route pages consume the Supabase browser client factory and typed `Thought` domain types from `src/lib/`
- The MCP proxy route injects server-side credentials from `MCP_URL`/`MCP_KEY` env vars sourced via `src/lib/`

#### `recipes/repo-learning-coach/src/` → `recipes/repo-learning-coach/server/`
- The React frontend calls the Express HTTP API server over localhost HTTP at runtime
- `src/lib/` provides the shared typed API client and TypeScript type definitions that define the contract between the two layers

#### `recipes/vercel-neon-telegram/src/app/api/` → `recipes/vercel-neon-telegram/src/lib/`
- `telegram/route.ts` imports `captureThought` (lib/capture), `searchThoughts` (lib/db), `generateEmbedding` (lib/ai)
- `mcp/route.ts` imports `requireAuth` (lib/auth), `checkRateLimit` (lib/rate-limit), `captureThought`, `searchThoughts`, `listThoughts`, `generateEmbedding`
- `capture/route.ts` imports `requireAuth`, `checkRateLimit`, `captureThought`

#### `.github/workflows/` → `.github/`
- `ob1-gate.yml` validates PR contributions against `.github/metadata.schema.json` at runtime
- `ob1-pr-followups.yml` downloads the structured gate artifact and conditionally invokes `anthropics/claude-code-action@v1`
- `claude-review.yml` (manual dispatch) also invokes `anthropics/claude-code-action@v1` with a broader tool allowlist

---

## External Dependencies

### Deno / Edge Function Packages

Used by: `server/`, all `extensions/` (family-calendar, home-maintenance, household-knowledge, job-hunt, meal-planning, professional-crm), `integrations/entity-extraction-worker`, `recipes/ob-graph`, `recipes/work-operating-model-activation`

| Package | Version | Purpose |
|---------|---------|---------|
| `@supabase/supabase-js` | npm:2.47.10 | Supabase Postgres client |
| `@modelcontextprotocol/sdk` | npm:1.24.3 | MCP protocol: server, tools, transport |
| `@hono/mcp` | npm:0.1.1 | Hono-based MCP transport adapter |
| `hono` | npm:4.9.2 | HTTP framework for Deno Edge Functions |
| `zod` | npm:4.1.13 | Runtime schema validation |

Used by: `integrations/kubernetes-deployment` only

| Package | Version | Purpose |
|---------|---------|---------|
| `postgres` (deno.land/x) | 0.19.3 | Direct Postgres connection for non-Supabase deployment |

### Node.js Runtime Dependencies — dashboards/open-brain-dashboard-next

| Package | Version | Purpose |
|---------|---------|---------|
| `next` | 16.2.1 | Next.js 14 application framework |
| `react` | 19.2.4 | UI rendering |
| `react-dom` | 19.2.4 | DOM bindings for React |
| `iron-session` | ^8.0.4 | Encrypted HTTP-only cookie session management |
| `server-only` | ^0.0.1 | Compile-time guard: prevents server modules leaking into client bundle |
| `@dnd-kit/core` | ^6.3.1 | Drag-and-drop primitives (kanban board) |
| `@dnd-kit/sortable` | ^10.0.0 | Sortable drag-and-drop layer |
| `@dnd-kit/utilities` | ^3.2.2 | DnD Kit utility helpers |

### Node.js Dev Dependencies — dashboards/open-brain-dashboard-next

| Package | Version | Purpose |
|---------|---------|---------|
| `tailwindcss` | ^4 | Utility-first CSS framework |
| `@tailwindcss/postcss` | ^4 | PostCSS integration for Tailwind v4 |
| `typescript` | ^5 | TypeScript compiler |
| `eslint` | ^9 | Linter |
| `eslint-config-next` | 16.2.1 | Next.js ESLint rule preset |
| `@types/node` | ^20 | Node.js type definitions |
| `@types/react` | ^19 | React type definitions |
| `@types/react-dom` | ^19 | React DOM type definitions |

### Node.js Dependencies — dashboards/open-brain-dashboard (SvelteKit)

All listed under `devDependencies` per SvelteKit convention (all packages are build-time bundled):

| Package | Version | Purpose |
|---------|---------|---------|
| `@sveltejs/kit` | ^2.50.2 | SvelteKit application framework |
| `svelte` | ^5.51.0 | Svelte 5 component compiler (runes mode) |
| `@supabase/supabase-js` | ^2.56.0 | Supabase browser client |
| `@supabase/ssr` | ^0.6.1 | Supabase SSR auth helpers |
| `tailwindcss` | ^4.2.1 | Utility-first CSS framework |
| `@tailwindcss/vite` | ^4.2.1 | Vite plugin for Tailwind v4 |
| `vite` | ^7.3.1 | Build tool and dev server |
| `@sveltejs/vite-plugin-svelte` | ^6.2.4 | Svelte Vite integration |
| `@sveltejs/adapter-vercel` | ^6.3.3 | Vercel deployment adapter |
| `@sveltejs/adapter-auto` | ^7.0.0 | Auto-detect deployment adapter |
| `svelte-check` | ^4.4.2 | Svelte TypeScript type checking |
| `typescript` | ^5.9.3 | TypeScript compiler |

### Node.js Runtime Dependencies — recipes/repo-learning-coach

| Package | Version | Purpose |
|---------|---------|---------|
| `express` | ^5.2.1 | HTTP API server |
| `@supabase/supabase-js` | ^2.47.10 | Supabase client for lesson/thought persistence |
| `react` | ^19.2.0 | Frontend UI framework |
| `react-dom` | ^19.2.0 | DOM bindings for React |
| `react-markdown` | ^10.1.0 | Markdown rendering in the lesson UI |
| `gray-matter` | ^4.0.3 | Frontmatter parsing for curriculum markdown files |
| `zod` | ^4.1.13 | Schema validation |

### Node.js Dev Dependencies — recipes/repo-learning-coach

| Package | Version | Purpose |
|---------|---------|---------|
| `vite` | ^7.1.7 | Frontend build tool |
| `@vitejs/plugin-react` | ^5.0.4 | React fast-refresh for Vite |
| `tsx` | ^4.21.0 | TypeScript execution for Node (server runner, sync script) |
| `typescript` | ~5.9.3 | TypeScript compiler |
| `concurrently` | ^9.2.1 | Parallel dev server runner (`npm run dev`) |
| `eslint` | ^9.39.1 | Linter |
| `eslint-plugin-react-hooks` | ^7.0.1 | React Hooks lint rules |
| `eslint-plugin-react-refresh` | ^0.4.24 | React Refresh lint rules |
| `@types/express` | ^5.0.5 | Express type definitions |
| `@types/node` | ^24.6.0 | Node.js type definitions |
| `@types/react` | ^19.2.2 | React type definitions |
| `@types/react-dom` | ^19.2.2 | React DOM type definitions |
| `typescript-eslint` | ^8.46.0 | TypeScript ESLint integration |
| `globals` | ^16.4.0 | Global variable definitions for ESLint |

### Node.js Runtime Dependencies — recipes/vercel-neon-telegram

| Package | Version | Purpose |
|---------|---------|---------|
| `next` | ^15.3.0 | Next.js application framework |
| `react` | ^19.1.0 | UI rendering |
| `react-dom` | ^19.1.0 | DOM bindings |
| `@ai-sdk/openai` | ^1.3.0 | AI SDK OpenAI provider (embeddings) |
| `ai` | ^4.3.0 | Vercel AI SDK core |
| `@modelcontextprotocol/sdk` | ^1.12.1 | MCP protocol implementation |
| `@neondatabase/serverless` | ^1.0.0 | Neon Postgres serverless driver |
| `grammy` | ^1.35.0 | Telegram Bot API framework |
| `zod` | ^3.24.0 | Schema validation |

### Node.js Dev Dependencies — recipes/vercel-neon-telegram

| Package | Version | Purpose |
|---------|---------|---------|
| `tsx` | ^4.19.0 | TypeScript execution for migration and utility scripts |
| `typescript` | ^5.8.0 | TypeScript compiler |
| `vitest` | ^4.1.0 | Unit test runner |
| `@types/node` | ^22.15.0 | Node.js type definitions |
| `@types/react` | ^19.1.0 | React type definitions |

### Node.js Dependencies — Single-Script Import Recipes

| Recipe | Declared npm Dependencies | Node Built-ins Used |
|--------|--------------------------|---------------------|
| `recipes/fingerprint-dedup-backfill` | none | `node:crypto` (SHA-256 fingerprinting) |
| `recipes/google-activity-import` | none | `fetch` (Supabase REST) |
| `recipes/grok-export-import` | `@supabase/supabase-js ^2.49.0`, `dotenv ^16.4.0` | `fetch` |
| `recipes/instagram-import` | `@supabase/supabase-js ^2.49.0`, `dotenv ^16.4.0` | `fetch` |
| `recipes/journals-blogger-import` | `@supabase/supabase-js ^2.49.0`, `dotenv ^16.4.0` | XML parsing (DOMParser) |
| `recipes/x-twitter-import` | `@supabase/supabase-js ^2.49.0`, `dotenv ^16.4.0` | `fetch` |

### Python Dependencies — Import Recipes

| Package | Version | Used By | Purpose |
|---------|---------|---------|---------|
| `requests` | >=2.28 | chatgpt-conversation-import, local-ollama-embeddings, obsidian-vault-import, perplexity-conversation-import | HTTP client for Supabase REST API and Ollama |
| `python-frontmatter` | >=1.0 | obsidian-vault-import | Markdown frontmatter parsing |
| `openpyxl` | >=3.1 | perplexity-conversation-import | Reading `.xlsx` Perplexity export files |

### GitHub Actions — External Actions

| Action | Version | Used In | Purpose |
|--------|---------|---------|---------|
| `actions/checkout` | v4 | ob1-gate.yml, ob1-pr-followups.yml, markdown-lint.yml, claude-review.yml | Repository checkout |
| `actions/upload-artifact` | v4 | ob1-gate.yml | Upload structured gate review artifact |
| `actions/github-script` | v7 | ob1-pr-followups.yml, auto-label.yml, welcome-new-contributors.yml | GitHub API scripting in workflow steps |
| `actions/setup-node` | v4 | markdown-lint.yml | Node.js setup for markdownlint-cli2 |
| `anthropics/claude-code-action` | v1 | ob1-pr-followups.yml, claude-review.yml, claude-issue-triage.yml | LLM-backed PR quality review and issue triage |
| `release-drafter/release-drafter` | v6 | release-drafter.yml | Automated release changelog drafting |

### External Services and Platforms (Runtime)

| Service | Used By | Purpose |
|---------|---------|---------|
| Supabase (hosted) | server/, all extensions, most recipes, both dashboards, entity-extraction-worker | Postgres + pgvector database, Edge Function hosting, auth |
| OpenRouter API | server/ (embeddings + metadata LLM), entity-extraction-worker (provider fallback chain), repo-learning-coach (brain bridge embeddings) | Multi-provider LLM and embedding gateway |
| Anthropic Claude API | entity-extraction-worker, adaptive-capture-classification, entity-wiki, infographic-generator, panning-for-gold (recipe), thought-enrichment, typed-edge-classifier, wiki-synthesis, wiki-compiler, skills/claudeception, skills/panning-for-gold | Named entity extraction, classification, enrichment, synthesis |
| OpenAI API | recipes/vercel-neon-telegram/src/lib/ | Embeddings via Vercel AI SDK (`@ai-sdk/openai`) |
| Neon Postgres | recipes/vercel-neon-telegram | Alternative Postgres backend for non-Supabase deployments |
| Telegram Bot API | recipes/vercel-neon-telegram | Thought capture and retrieval over Telegram |
| Gmail API / Google OAuth2 | recipes/email-history-import | Gmail email history import |
| Ollama (local) | recipes/local-ollama-embeddings | Locally-run embedding generation (no cloud API) |
| Kubernetes / Docker | integrations/kubernetes-deployment | Self-hosted MCP server deployment target |
| Vercel | dashboards/open-brain-dashboard (adapter-vercel), recipes/vercel-neon-telegram | Frontend hosting platform |
| GitHub Actions runner | .github/workflows/ | CI/CD workflow execution environment |

---

## Circular Dependencies

None detected.

---

## Dependency Health Notes

- `schemas/typed-reasoning-edges` has a hard runtime prerequisite on `schemas/entity-extraction`. Applying these migrations out of order raises an exception with a descriptive message.
- `integrations/entity-extraction-worker/_shared/` is duplicated by copy into each Edge Function directory. This is required by Supabase Edge Function deployment constraints (functions cannot share code across directories at deploy time).
- `recipes/fingerprint-dedup-backfill` and `recipes/google-activity-import` declare no npm dependencies and rely entirely on Node.js built-in modules plus `fetch` for Supabase REST calls.
- `dashboards/open-brain-dashboard` (SvelteKit) lists all packages under `devDependencies` following SvelteKit convention; there are no separate runtime npm packages.
- `recipes/vercel-neon-telegram` pins `zod` at `^3.24.0` while other modules in the repo target `zod` 4.x (`npm:zod@4.1.13` in Deno manifests, `^4.1.13` in repo-learning-coach). These are isolated per-recipe installs and do not conflict.
- The `server/` module uses `NEXT_PUBLIC_API_URL` is a naming note only: despite the `NEXT_PUBLIC_` prefix on the dashboard env var, the server URL is consumed exclusively server-side in `dashboards/open-brain-dashboard-next/lib/api.ts`.
