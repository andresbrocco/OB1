# ARCHITECTURE.md

> System design facts for this codebase.

## System Type

**Monorepo of composable components built on a shared MCP + Supabase/pgvector memory core.**

Open Brain (OB1) is not a single deployable application. It is a community repository that ships:

- One canonical **MCP server** (a Deno/Supabase Edge Function) as the central memory backend
- A **database schema layer** of additive SQL migrations that extend the core `thoughts` table
- Multiple **dashboard frontends** (Next.js, SvelteKit) deployed as separate Vercel applications
- **Extensions**: curated, domain-specific MCP servers deployed as additional Supabase Edge Functions
- **Integrations**: async background workers and alternative deployment targets (Kubernetes)
- **Recipes**: standalone capability scripts and applications that compose against the core
- **Skills**: AI behavioral prompt packs with no deployment infrastructure
- **Primitives**: reusable concept guides for shared infrastructure patterns
- **CI/governance**: GitHub Actions workflows that automate the contribution lifecycle

All components share a single protocol boundary: the Model Context Protocol (MCP) over HTTP, with `x-brain-key` access-key authentication. The core `thoughts` table in Supabase is the single source of truth for all memory data.

---

## Component Overview

| Component | Responsibility | Key Modules |
|-----------|---------------|-------------|
| Core MCP Server | Thought capture, semantic search, listing, stats — the primary memory API | `server/` |
| Schema Extensions | Additive SQL migrations: knowledge graph, enhanced classification, typed edges | `schemas/entity-extraction`, `schemas/enhanced-thoughts`, `schemas/typed-reasoning-edges` |
| Extensions | Domain-specific MCP servers (calendar, CRM, meal planning, etc.) as additional Edge Functions | `extensions/family-calendar`, `extensions/professional-crm`, `extensions/meal-planning`, `extensions/job-hunt`, `extensions/home-maintenance`, `extensions/household-knowledge` |
| Integrations | Async workers and capture connectors | `integrations/entity-extraction-worker`, `integrations/kubernetes-deployment` |
| Next.js Dashboard | Full-featured browser UI — thought browse, search, kanban, audit, reflections | `dashboards/open-brain-dashboard-next` |
| SvelteKit Dashboard | Alternative browser UI via MCP proxy | `dashboards/open-brain-dashboard` |
| Recipes | Standalone capability builds: importers, enrichment pipelines, graph generators | `recipes/*` |
| Skills | AI behavioral prompt packs loaded into AI client context | `skills/*` |
| Primitives | Concept guides for shared infrastructure patterns | `primitives/*` |
| CI / Governance | PR gate, LLM review, contributor automation | `.github/`, `.github/workflows/` |
| Docs | User-facing setup guides, companion prompts, FAQ | `docs/` |

---

## Layer Structure

The system has no single unified layering. Components follow distinct patterns depending on their role.

### Core Memory Layer (server/)

The foundational layer. Exposes the `thoughts` table over MCP.

- Handles: MCP tool definitions, HTTP transport, `x-brain-key` auth, embedding generation (OpenRouter), metadata extraction (LLM), and thought persistence via `upsert_thought` / `match_thoughts` Supabase RPCs
- Deployed as: Supabase Edge Function (Deno runtime)
- Entry point: `Deno.serve`

### Database Schema Layer (schemas/)

SQL migrations applied on top of the core Supabase database.

- `enhanced-thoughts`: Adds classification columns (`type`, `sensitivity_tier`, `importance`, `quality_score`, `source_type`, `enriched`) and utility RPCs (`search_thoughts_text`, `brain_stats_aggregate`, `get_thought_connections`)
- `entity-extraction`: Adds knowledge graph tables (`entities`, `edges`, `thought_entities`, `entity_extraction_queue`, `consolidation_log`) and a trigger that auto-enqueues thoughts for async entity extraction
- `typed-reasoning-edges`: Adds `thought_edges` table for typed semantic relations between thoughts; also adds temporal validity columns to the `edges` table from entity-extraction

**Apply order is enforced**: `entity-extraction` requires `thoughts.content_fingerprint` (core setup Step 2.6). `typed-reasoning-edges` requires `entity-extraction` to be applied first.

### Extension Layer (extensions/)

Six curated MCP servers, each a Deno/Supabase Edge Function, adding domain-specific memory tools.

- Each extension pairs a `schema.sql` migration with an `index.ts` Edge Function
- All use `DEFAULT_USER_ID` + service role key (single-tenant, RLS bypassed by default)
- One exception: `professional-crm` requires the `rls` primitive for per-user isolation
- Entry pattern: new `McpServer` + `StreamableHTTPTransport` instantiated per request (stateless)

### Integration Layer (integrations/)

Background workers and capture connectors.

- `entity-extraction-worker`: Deno Edge Function, cron-invoked, drains `entity_extraction_queue`, calls LLM, writes to knowledge graph tables. Uses `_shared/` (config.ts, helpers.ts) copied per Edge Function.
- `kubernetes-deployment`: Self-hosted variant replacing Supabase with direct PostgreSQL+pgvector. Long-lived Deno/Hono HTTP server, not a serverless function. An explicit exception to the repo's remote-MCP-only rule.

### Dashboard Layer (dashboards/)

Two independent frontend applications, each connecting to the backend via different protocols.

**`open-brain-dashboard-next` (Next.js 14):**
- `components/`: Client-side React UI
- `app/api/`: Next.js Route Handlers — authenticated proxy layer; enforces session auth; applies auto-routing heuristics for multi-thought ingest
- `lib/`: Server-only API client (`api.ts`, marked `"server-only"`), iron-session auth helpers, domain types
- Connects to: Open Brain REST API (`NEXT_PUBLIC_API_URL` → Supabase Edge Function)
- Auth: iron-session encrypted HTTP-only cookie storing `x-brain-key`

**`open-brain-dashboard` (SvelteKit 5):**
- `src/routes/`: Page components and server-side MCP proxy route (`/api/mcp`)
- `src/lib/`: Supabase browser client, typed API client, MCP text-response parsers
- Connects to: MCP server directly via JSON-RPC 2.0 proxy route
- Auth: Supabase Auth (anon key + OAuth) + private env vars for MCP credentials

### Recipe Layer (recipes/)

Standalone capability builds. Each recipe is an independent script or application. Notable sub-architectures:

- `vercel-neon-telegram`: Full alternative deployment stack (Next.js + Neon Postgres + Telegram bot)
- `repo-learning-coach`: Express HTTP API + React frontend; syncs markdown curriculum to Supabase on startup; bridges lessons to Open Brain via `match_thoughts` / `upsert_thought`
- `wiki-compiler`: Multi-phase pipeline orchestrator sequencing entity-extraction, typed-edge-classification, entity-wiki generation, and topic wiki synthesis
- Single-script importers (ChatGPT, Gmail, Obsidian, Instagram, Twitter, Grok, Perplexity, Journals/Blogger): standalone Node.js or Python scripts writing to the `thoughts` table via Supabase REST

### Skill Layer (skills/)

Prompt packs loaded into an AI client's context. No deployment infrastructure.

- Skills reference MCP tools (e.g., `capture_thought`, `search_thoughts`) by soft name; connector prefix varies by environment
- Some skills (claudeception/aiception, panning-for-gold) have self-improvement loops that update the skill file itself after sessions
- `heavy-file-ingestion` is the only skill with executable scripts (`scripts/`)

### CI / Governance Layer (.github/)

Two-stage PR review pipeline:

1. `ob1-gate.yml`: Deterministic shell-script structural and safety checks; uploads artifact
2. `ob1-pr-followups.yml`: Downloads artifact, posts idempotent PR comment, conditionally invokes Claude (`claude-code-action`) for qualitative review

Additional workflows: issue triage, contributor onboarding, labeling, markdown linting, Discord announcements, release drafting.

---

## Data Flow

### Core Thought Capture Flow

```
AI Client (Claude Desktop, ChatGPT, etc.)
    │
    │  MCP HTTP POST (x-brain-key header or ?key= param)
    ▼
server/ (Supabase Edge Function)
    │
    ├─► OpenRouter API ──► embedding (text-embedding-3-small)
    ├─► OpenRouter API ──► metadata extraction (gpt-4o-mini)   [parallel via Promise.all]
    │
    ├─► Supabase RPC: upsert_thought (content-level dedup)
    └─► Supabase UPDATE: patch embedding vector
```

### Async Knowledge Graph Enrichment Flow

```
thoughts table INSERT/UPDATE
    │
    ▼ (database trigger: trg_queue_entity_extraction)
entity_extraction_queue (status: pending)
    │
    ▼ (cron invocation)
integrations/entity-extraction-worker
    │
    ├─► LLM (OpenRouter → OpenAI → Anthropic fallback chain)
    │
    ├─► entities table (upsert canonical entities)
    ├─► edges table (increment support_count)
    └─► thought_entities table (link thoughts to entities)
```

### Dashboard (Next.js) Request Flow

```
Browser
    │
    ▼
middleware.ts (cookie presence check → redirect to /login)
    │
    ▼
app/api/* Route Handler
    ├─► requireSession() (session validation, reads x-brain-key from iron-session cookie)
    │
    ▼
lib/api.ts (server-only, apiFetch with x-brain-key header)
    │
    ▼
Open Brain REST API (Supabase Edge Function at NEXT_PUBLIC_API_URL)
    │
    ▼
Supabase (thoughts table, pgvector search, RPCs)
```

### Dashboard (SvelteKit) Request Flow

```
Browser
    │
    ▼
+layout.server.ts (auth guard: locals.user check → redirect to /signin)
    │
    ▼
src/routes/api/mcp/+server.ts (MCP proxy, injects MCP_URL + MCP_KEY from private env)
    │
    ▼  JSON-RPC 2.0
MCP Server (Supabase Edge Function)
    │
    ▼
Supabase (thoughts table)
```

### Entity-Extraction → Knowledge Graph → Wiki Pipeline

```
thoughts table
    ▼ (trigger)
entity_extraction_queue
    ▼ (entity-extraction-worker cron)
entities / edges / thought_entities
    ▼ (recipes/typed-edge-classifier)
thought_edges (typed semantic reasoning relations)
    ▼ (recipes/entity-wiki)
Entity wiki pages (file / entity metadata / dossier thought)
    ▼ (recipes/wiki-synthesis or recipes/wiki-compiler)
Compiled wiki articles (markdown files)
```

### Key Data Paths

| Flow | Path | Description |
|------|------|-------------|
| Thought capture | AI Client → server/ → OpenRouter → Supabase | Embeds, extracts metadata, upserts thought |
| Semantic search | AI Client → server/ → Supabase match_thoughts RPC | Vector cosine similarity via pgvector |
| Knowledge graph build | thoughts → trigger → queue → entity-extraction-worker → LLM → entities/edges | Async, cron-driven |
| Typed edge classification | thought_entities → typed-edge-classifier → Haiku filter → Opus classify → thought_edges | Two-stage LLM pipeline |
| Dashboard thought browse | Browser → Next.js API routes → lib/api.ts → Open Brain REST API → Supabase | Server-side proxied |
| Telegram capture | Telegram webhook → vercel-neon-telegram /api/telegram → Neon Postgres | Alternative deployment stack |
| Lesson capture | repo-learning-coach UI → Express server → brain bridge → upsert_thought RPC | Local recipe, optional brain bridge |

---

## External Boundaries

### APIs Exposed

| API | Transport | Location | Purpose |
|-----|-----------|----------|---------|
| MCP server | HTTP + StreamableHTTP (Supabase Edge Function) | `server/` | Core memory: capture, search, list, stats |
| Extension MCP servers (×6) | HTTP + StreamableHTTP (Supabase Edge Functions) | `extensions/*/index.ts` | Domain-specific memory tools |
| Entity extraction worker | HTTP (Supabase Edge Function) | `integrations/entity-extraction-worker/` | Async knowledge graph population |
| Next.js API routes | HTTP (Vercel serverless) | `dashboards/open-brain-dashboard-next/app/api/` | Browser dashboard backend |
| SvelteKit MCP proxy | HTTP (Vercel serverless) | `dashboards/open-brain-dashboard/src/routes/api/mcp/` | Browser dashboard MCP relay |
| vercel-neon-telegram API | HTTP (Vercel serverless) | `recipes/vercel-neon-telegram/src/app/api/` | REST capture, MCP, Telegram webhook |
| repo-learning-coach server | HTTP (Express, local) | `recipes/repo-learning-coach/server/` | Learning coach REST API |
| Kubernetes MCP server | HTTP (Hono, self-hosted) | `integrations/kubernetes-deployment/` | Self-hosted MCP server variant |

### Authentication Mechanisms

| Mechanism | Used By | Notes |
|-----------|---------|-------|
| `x-brain-key` header or `?key=` query param | server/, extensions/, kubernetes-deployment | Dual-mode: header for API clients, query param for Claude Desktop / ChatGPT |
| `Authorization: Bearer` header | vercel-neon-telegram | Alternative bearer format; same key extraction logic |
| iron-session encrypted cookie | open-brain-dashboard-next | Stores `x-brain-key` server-side; never sent to browser |
| Supabase Auth (anon + OAuth) | open-brain-dashboard (SvelteKit) | Session managed via `@supabase/ssr` |
| `x-telegram-bot-api-secret-token` | vercel-neon-telegram /api/telegram | Separate from brain key; set at Telegram webhook registration |

### External Services Consumed

| Service | Purpose | Consuming Modules |
|---------|---------|-------------------|
| Supabase (hosted) | Postgres + pgvector database, Edge Function hosting, auth | server/, all extensions, most recipes, both dashboards, entity-extraction-worker |
| OpenRouter API | Embeddings (`text-embedding-3-small`) + LLM metadata extraction (`gpt-4o-mini`); primary provider | server/, entity-extraction-worker, repo-learning-coach brain bridge |
| Anthropic Claude API | Named entity extraction, classification, enrichment, synthesis | entity-extraction-worker, adaptive-capture-classification, entity-wiki, infographic-generator, panning-for-gold recipe, thought-enrichment, typed-edge-classifier, wiki-synthesis, wiki-compiler, skills/claudeception, skills/panning-for-gold |
| OpenAI API | Embeddings via Vercel AI SDK | vercel-neon-telegram/src/lib/ |
| Neon Postgres | Alternative Postgres backend with pgvector | vercel-neon-telegram |
| Telegram Bot API | Thought capture and retrieval over Telegram | vercel-neon-telegram |
| Gmail API / Google OAuth2 | Email history import | recipes/email-history-import |
| Ollama (local) | Locally-run embedding generation (no cloud API) | recipes/local-ollama-embeddings |
| Kubernetes / Docker | Self-hosted MCP server deployment | integrations/kubernetes-deployment |
| Vercel | Frontend hosting | dashboards/open-brain-dashboard (adapter-vercel), vercel-neon-telegram |
| GitHub Actions runner | CI/CD workflow execution | .github/workflows/ |
| anthropics/claude-code-action | LLM-backed PR quality review and issue triage | .github/workflows/ (ob1-pr-followups.yml, claude-review.yml) |

---

## Deployment Units

Each component is independently deployed. There is no single build artifact.

| Unit | Components | Runtime | Deployment Target |
|------|------------|---------|-------------------|
| Core MCP server | `server/` | Deno Edge Function | Supabase Functions (`supabase functions deploy`) |
| Extension MCP servers (×6) | `extensions/*/index.ts` | Deno Edge Function | Supabase Functions (one deploy per extension) |
| Entity extraction worker | `integrations/entity-extraction-worker/` | Deno Edge Function | Supabase Functions (cron-invoked) |
| Next.js dashboard | `dashboards/open-brain-dashboard-next/` | Node.js 18+ | Vercel (or any Next.js host) |
| SvelteKit dashboard | `dashboards/open-brain-dashboard/` | Node.js 22.x | Vercel (adapter-vercel) |
| Vercel + Neon + Telegram | `recipes/vercel-neon-telegram/` | Node.js (Next.js App Router) | Vercel |
| Repo learning coach | `recipes/repo-learning-coach/` | Node.js (Express + Vite) | Local / self-hosted |
| Kubernetes MCP server | `integrations/kubernetes-deployment/` | Deno long-lived process | Kubernetes StatefulSet (single replica) |
| Supabase database | Core `thoughts` table + schema extensions | PostgreSQL + pgvector | Supabase managed service |
| CI workflows | `.github/workflows/` | GitHub Actions runner | GitHub (triggered on PR/push) |

### Schema Apply Order

When applying schema extensions to the core database, the required order is:

```
docs/01-getting-started.md Step 2.6   (adds thoughts.content_fingerprint — prerequisite for all schemas)
    ▼
schemas/enhanced-thoughts/schema.sql   (classification columns, full-text search RPCs)
    ▼
schemas/entity-extraction/schema.sql   (knowledge graph tables, extraction queue trigger)
    ▼
schemas/typed-reasoning-edges/schema.sql  (thought_edges table; adds temporal columns to edges)
```

Applying `entity-extraction` before Step 2.6 raises a hard exception. Applying `typed-reasoning-edges` before `entity-extraction` raises a hard exception.

---

## Ambiguities

- The `open-brain-dashboard-next` references an `open-brain-rest` Supabase Edge Function as its backend (`NEXT_PUBLIC_API_URL`). This REST API Edge Function is not present as a module in this repo; it is referenced only by URL in dashboard documentation.
- `integrations/slack-capture` and `integrations/discord-capture` are listed in the repo structure but no CONTEXT.md files exist for them; their implementation details are not documented in the analyzed files.
- The LLM provider fallback order in `integrations/entity-extraction-worker/_shared/` (OpenRouter → OpenAI → Anthropic) differs from some recipe implementations that call Anthropic directly without a fallback chain.
