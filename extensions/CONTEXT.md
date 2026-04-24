# CONTEXT.md — Extensions

## Purpose

Extensions are curated, domain-specific MCP servers that add structured memory capabilities to Open Brain. Each extension pairs a PostgreSQL schema (new tables) with a Supabase Edge Function that exposes those tables as MCP tools, allowing AI clients to read and write domain data through the same protocol as the core brain.

## Responsibility Boundaries

- **Owns**: The contract for what an extension is — 5 required files (`README.md`, `metadata.json`, `schema.sql`, `index.ts`, `deno.json`), the curated learning path ordering, and the validation checklist enforced by `_template/AGENT_SPEC.md`
- **Delegates to**: `primitives/deploy-edge-function` and `primitives/remote-mcp` for deployment and connector setup; `schemas/` for core table definitions
- **Does not handle**: The core `thoughts` table (never modified here), authentication flows (handled upstream by Supabase Auth), or LLM orchestration logic

## Key Concepts

- **Learning path**: Six curated extensions carry `learning_order` values 1–6 in `metadata.json`. These are the official progression from beginner to advanced and are **maintainer-gated** — no community contributions can join this set without approval. Community extensions omit `learning_order`.
- **`DEFAULT_USER_ID` pattern**: Extensions use the Supabase service role key (which bypasses Row Level Security) and scope all queries to a `DEFAULT_USER_ID` environment variable. This is a deliberate single-tenant simplification — the service role key means RLS is not enforced unless explicitly added (see `professional-crm`, which requires the `rls` primitive).
- **`_template/AGENT_SPEC.md`**: A machine-readable spec that AI agents use to generate new extensions in a single pass. It defines exact file structure, tool registration patterns, naming conventions, and a validation checklist.

## Non-Obvious Details

- Each `index.ts` instantiates a new `McpServer` and `StreamableHTTPTransport` **per request**, not once at module load. This is required by the stateless Edge Function execution model.
- The `family-calendar` extension includes a workaround for a Claude Desktop bug where connectors omit the `Accept: text/event-stream` header; it manually patches the request before passing it to `StreamableHTTPTransport`.
- Extensions that require the `rls` primitive (e.g., `professional-crm`) must explicitly list it in `requires_primitives`. Extensions that do not list `rls` are implicitly single-user and rely on `DEFAULT_USER_ID` instead of `auth.uid()`.
- `meal-planning` includes a `shared-server.ts` file, indicating it uses the `shared-mcp` primitive pattern for splitting tool logic from the Hono entry point — the only extension currently doing this.

## Related Modules

- **[.claude/skills](../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Skill file format)
- **[dashboards](../dashboards/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), Per-request MCP server instantiation)
- **[dashboards/open-brain-dashboard](../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP proxy route, Per-request MCP server instantiation)
- **[dashboards/open-brain-dashboard-next/lib](../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pattern, server-only boundary)
- **[dashboards/open-brain-dashboard/src](../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, Per-request MCP server instantiation)
- **[dashboards/open-brain-dashboard/src/lib](../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, Per-request MCP server instantiation)
- **[docs](../docs/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP tool context overhead, Per-request MCP server instantiation)
- **[extensions/family-calendar](family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, Per-request MCP server instantiation)
- **[extensions/home-maintenance](home-maintenance/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, DEFAULT_USER_ID single-tenant pattern)
- **[extensions/household-knowledge](household-knowledge/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept-header patch for Claude Desktop compatibility, Per-request MCP server instantiation)
- **[extensions/meal-planning](meal-planning/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, Per-request stateless MCP server instantiation)
- **[extensions/professional-crm](professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, Stateless MCP transport)
- **[integrations](../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, Per-request MCP server instantiation)
- **[recipes](../recipes/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/claudeception](../recipes/claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Skill Lifecycle)
- **[recipes/email-history-import](../recipes/email-history-import/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pattern, Two-layer dedup)
- **[recipes/entity-wiki](../recipes/entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Output modes (file / entity-metadata / thought))
- **[recipes/infographic-generator](../recipes/infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Manifest file)
- **[recipes/life-engine](../recipes/life-engine/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pattern, user_id as channel chat_id)
- **[recipes/research-to-decision-workflow](../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Prompt stubs)
- **[recipes/vercel-neon-telegram](../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Per-request MCP server instantiation, stateless MCP server)
- **[recipes/vercel-neon-telegram/src/app/api](../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP protocol compliance (HEAD/DELETE/OPTIONS), Per-request MCP server instantiation, Stateless MCP server per request)
- **[recipes/wiki-compiler](../recipes/wiki-compiler/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Compile manifest)
- **[recipes/wiki-synthesis](../recipes/wiki-synthesis/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Resume-safe JSONL state log)
- **[server](../server/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP tool registration, Per-request MCP server instantiation, StreamableHTTPTransport)
- **[skills](../skills/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Output Contract)
- **[skills/claudeception](../skills/claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Dual-scope skill storage, Learning path (learning_order 1-6), Skill lifecycle (creation to archival))
- **[skills/heavy-file-ingestion](../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Output directory convention (.ob1/))
- **[skills/heavy-file-ingestion/scripts](../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, AGENT_SPEC.md machine-readable generation spec)
- **[skills/panning-for-gold](../skills/panning-for-gold/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Permanent file write discipline)
- **[skills/work-operating-model](../skills/work-operating-model/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Canonical entry contract, Learning path (learning_order 1-6))
