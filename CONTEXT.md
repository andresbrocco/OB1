# CONTEXT.md

> Entry point for understanding this codebase.

## Project Overview

Open Brain (OB1) is a persistent AI memory system built on a single Supabase/pgvector database exposed over the Model Context Protocol (MCP). It is not a single deployable application — it is a community monorepo that ships a canonical MCP server, additive database schema extensions, domain-specific extension servers, frontend dashboards, background integrations, standalone capability recipes, and AI behavioral skill packs.

The core contract: one `thoughts` table in Supabase is the single source of truth. All components read from or write to it via MCP over HTTP, authenticated by a shared `x-brain-key` access key. Any MCP-compatible AI client (Claude Desktop, ChatGPT, etc.) can capture and retrieve memories using this protocol.

## Technology Stack

| Category | Technology |
|----------|------------|
| Core runtime | Deno (Supabase Edge Functions) |
| MCP protocol | `@modelcontextprotocol/sdk` + `@hono/mcp` StreamableHTTP transport |
| Database | Supabase (PostgreSQL + pgvector) |
| Embeddings | OpenRouter (`text-embedding-3-small`) |
| LLM metadata | OpenRouter (`gpt-4o-mini`); Anthropic Claude for enrichment recipes |
| Next.js dashboard | Next.js 14, React 19, iron-session, Tailwind CSS v4 |
| SvelteKit dashboard | SvelteKit 5 (Svelte runes), Supabase Auth, Tailwind CSS v4 |
| Schema validation | Zod (Deno + Node.js) |
| Testing | vitest (recipes/vercel-neon-telegram only) |
| CI/CD | GitHub Actions |

## Quick Orientation

### Entry Points

| Entry | Path | Purpose |
|-------|------|---------|
| Core MCP Server | `server/` | Supabase Edge Function — thought capture, semantic search, stats |
| Extensions (×6) | `extensions/*/index.ts` | Domain-specific MCP servers (calendar, CRM, meal planning, etc.) |
| Entity Extraction Worker | `integrations/entity-extraction-worker/` | Cron-driven knowledge graph population |
| Next.js Dashboard | `dashboards/open-brain-dashboard-next/` | Browser UI — browse, search, kanban, audit |
| SvelteKit Dashboard | `dashboards/open-brain-dashboard/` | Alternative browser UI via MCP proxy |

### Deploying the Core MCP Server

```bash
# Deploy to Supabase (requires Supabase CLI)
supabase functions deploy server

# Required secrets (set before deploying)
supabase secrets set SUPABASE_URL=...
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=...
supabase secrets set OPENROUTER_API_KEY=...
supabase secrets set MCP_ACCESS_KEY=...
```

### Running the Next.js Dashboard

```bash
cd dashboards/open-brain-dashboard-next
cp .env.example .env.local
# Fill in NEXT_PUBLIC_API_URL and SESSION_SECRET (32+ chars)
npm install
npm run dev
```

### Running Tests (vercel-neon-telegram only)

```bash
cd recipes/vercel-neon-telegram
npm test
```

## Documentation Map

- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — System design, component overview, data flows, and deployment units
- [DEPENDENCIES.md](docs/DEPENDENCIES.md) — Module relationships, external packages, and dependency graph
- [PATTERNS.md](docs/PATTERNS.md) — Auth patterns, error handling, MCP server structure, LLM provider fallback
- [TESTING.md](docs/TESTING.md) — Test organization and commands
- [SECURITY.md](docs/SECURITY.md) — Authentication, authorization, secrets management, known considerations
- [INFRASTRUCTURE.md](docs/INFRASTRUCTURE.md) — CI/CD workflows, Docker, Kubernetes, deployment instructions

## Module Overview

Key modules in this codebase:

| Module | Purpose | CONTEXT.md |
|--------|---------|------------|
| `server/` | Core MCP server: thought capture, semantic search, listing, stats | [→](server/CONTEXT.md) |
| `schemas/` | Additive SQL migrations: enhanced-thoughts, entity-extraction, typed-reasoning-edges | [→](schemas/CONTEXT.md) |
| `extensions/` | Six curated domain-specific MCP servers (curated learning path) | [→](extensions/CONTEXT.md) |
| `integrations/` | Entity extraction worker, Kubernetes self-hosted variant, capture connectors | [→](integrations/CONTEXT.md) |
| `dashboards/` | Next.js and SvelteKit browser UIs | [→](dashboards/CONTEXT.md) |
| `recipes/` | Standalone capability builds: importers, enrichment pipelines, graph generators | [→](recipes/CONTEXT.md) |
| `skills/` | AI behavioral prompt packs (no deployment infrastructure) | [→](skills/CONTEXT.md) |
| `.github/` | Two-stage PR governance: deterministic gate + LLM-backed qualitative review | [→](.github/CONTEXT.md) |

For a complete list of all submodules with their own CONTEXT.md files, see [ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Contribution Model

Contributions live in their own subfolder under the appropriate category directory. Every contribution requires `README.md` and `metadata.json`. The PR gate (`ob1-gate.yml`) enforces structural rules, metadata schema validation, credential scanning, and SQL safety checks automatically. See `CONTRIBUTING.md` for the full contribution contract.

**Schema migration apply order** (enforced by runtime prerequisite checks):

```
docs/01-getting-started.md Step 2.6  (adds thoughts.content_fingerprint)
    ↓
schemas/enhanced-thoughts/schema.sql
    ↓
schemas/entity-extraction/schema.sql
    ↓
schemas/typed-reasoning-edges/schema.sql
```
