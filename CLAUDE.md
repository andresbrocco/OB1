# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## Project Overview

Open Brain (OB1) is a persistent AI memory system built on a single Supabase/pgvector database exposed over the Model Context Protocol (MCP) — one database, one protocol, any MCP-compatible AI client. This repo is a community monorepo shipping a canonical MCP server, additive schema extensions, domain-specific extension servers, frontend dashboards, background integrations, standalone capability recipes, and AI behavioral skill packs, all sharing the `thoughts` table as the single source of truth.

## Environment

| What | Command |
|------|---------|
| Runtime | Deno (Edge Functions); Node.js 18+ (dashboards/recipes) |
| Deploy (core MCP) | `supabase functions deploy open-brain-mcp --no-verify-jwt` |
| Dev (Next.js dashboard) | `cd dashboards/open-brain-dashboard-next && npm run dev` |
| Dev (SvelteKit dashboard) | `cd dashboards/open-brain-dashboard && npm run dev` |
| Test | `cd recipes/vercel-neon-telegram && npm test` |

See [README.md](README.md) for full command reference.

## Finding Information

This codebase has **distributed documentation** at multiple directory levels:

| File | Audience | Contains |
|------|----------|----------|
| `CONTEXT.md` | AI agents | Architecture, boundaries, concepts, gotchas |
| `README.md` | Developers | Ports, env vars, commands, endpoints, troubleshooting |

**Before modifying any area**, find and read the nearest documentation:
```
glob "**/target-area/**/CONTEXT.md"
glob "**/target-area/**/README.md"
```

### Root Documentation

| Document | When to consult |
|----------|----------------|
| [CONTEXT.md](CONTEXT.md) | Starting point — project overview, module map |
| [README.md](README.md) | Quick start, all commands, ports, env vars |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Understanding system design or data flow |
| [docs/DEPENDENCIES.md](docs/DEPENDENCIES.md) | Module relationships, external packages |
| [docs/PATTERNS.md](docs/PATTERNS.md) | Code patterns, naming conventions to follow |
| [docs/TESTING.md](docs/TESTING.md) | Test structure, frameworks, how to run tests |
| [docs/SECURITY.md](docs/SECURITY.md) | Auth, validation, secrets handling |
| [docs/INFRASTRUCTURE.md](docs/INFRASTRUCTURE.md) | Docker, CI/CD, deployment |

## Architecture

OB1 is a monorepo of independently deployed composable components, not a single application. The core layer is a Deno/Supabase Edge Function MCP server (`server/`) that exposes thought capture, semantic search, listing, and stats over HTTP with `x-brain-key` access-key auth. Additive SQL schemas extend the core `thoughts` table with a knowledge graph, enhanced classification, and typed reasoning edges; an async entity extraction worker drains a Postgres-backed queue to populate that graph. Two independent dashboard frontends (Next.js 14 and SvelteKit 5) proxy all data access through server-side route handlers — the access key never reaches the browser. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for details.

## Key Conventions

- **Never modify the core `thoughts` table structure.** Adding columns is allowed; altering or dropping existing ones is not.
- **MCP servers must be remote (Supabase Edge Functions), never local.** Never use `StdioServerTransport`, `claude_desktop_config.json`, or local `mcpServers` patterns.
- **No credentials, API keys, or secrets in any file.** Always use environment variables; each component ships a `.env.example`.
- **No destructive SQL in any committed file.** No `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`, or unqualified `DELETE FROM`.
- **No binary blobs over 1 MB.** No `.exe`, `.dmg`, `.zip`, `.tar.gz`.
- **Every contribution lives in its own subfolder** under the correct category directory and must include both `README.md` and `metadata.json`.
- **MCP server instantiation is per-request**, not per module load — required by the stateless Edge Function execution model. Always create a new `McpServer` instance inside the request handler.
- **Use `zod` for all MCP tool input schemas and API route body parsing** in TypeScript components.
- **All environment variables are read at module load time.** Missing required config must throw immediately at startup, not silently at request time.
- **Sensitivity tier assignment is escalation-only** (`standard → personal → restricted`). Never downgrade a tier in pipeline code; unrecognized values normalize to `"personal"`.
- **Shared Edge Function utilities (`_shared/`) must be copied** into each function directory — Supabase cannot share code across function boundaries at deploy time. Apply security fixes to all copies.

See [docs/PATTERNS.md](docs/PATTERNS.md) for all patterns.

## Testing

Tests exist only in `recipes/vercel-neon-telegram`. The framework is **vitest** (`^4.1.0`); three unit test files cover auth key extraction, rate-limit window logic, and Zod schema validation. Run with `cd recipes/vercel-neon-telegram && npm test`. All other modules in this monorepo have no automated tests. See [docs/TESTING.md](docs/TESTING.md) for full testing guide.

## Security Considerations

- **All MCP and REST endpoints must validate `x-brain-key`** via three extraction strategies in order: `x-brain-key` header → `Authorization: Bearer` header → `?key=` query parameter. The query-param fallback is required for Claude Desktop and ChatGPT.
- **Use `timingSafeEqual` for access key comparison** in Node.js components to prevent timing-based enumeration.
- **The Next.js dashboard `lib/api.ts` is marked `server-only`** — the `x-brain-key` must never be included in any browser-side bundle.
- **Wrap thought content in XML delimiters before inserting into LLM prompts** and escape any literal occurrences of those delimiter strings to guard against prompt injection.
- **LLM-returned metadata must be sanitized** before writing to the database — clamp numeric fields to allowed ranges and constrain enum fields to explicit allowlists.
- **`SUPABASE_SERVICE_ROLE_KEY` bypasses RLS.** Only `extensions/professional-crm` implements per-user isolation. All other extensions and the core server are single-tenant by design.

See [docs/SECURITY.md](docs/SECURITY.md) for full security reference.

## Local GSD Execution Layer

This repo also has a maintainer-local GSD layer in `.planning/`.

- If `.planning/` exists, use it for local brownfield planning and phased execution.
- Start with `.planning/STATE.md`, then read `.planning/PROJECT.md`, `.planning/ROADMAP.md`, and the relevant `.planning/codebase/*.md` documents.
- Keep `.planning/` local. It is gitignored intentionally and is not part of the public contribution contract or upstream PR scope.
- Public contributor rules still come from `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, and the committed repo files.
