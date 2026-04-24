# PATTERNS.md

> Observed code patterns in this codebase.

## Authentication & Authorization

How access control is implemented across the system.

### Pattern

A single shared secret (`x-brain-key`) is the primary authentication mechanism for all MCP servers and REST API endpoints. Every endpoint that accepts external calls supports three key-extraction strategies in priority order: the `x-brain-key` custom header, an `Authorization: Bearer <key>` header, and a `?key=` query parameter. The query-parameter fallback is intentional to support AI clients (Claude Desktop, ChatGPT) that cannot send custom headers.

Key comparison uses `timingSafeEqual` (Node.js `crypto` module) in Node-based components to prevent timing-attack-based key extraction.

### Dashboard Authentication

The Next.js dashboard adds a second layer on top of the `x-brain-key` pattern:

- **iron-session** encrypted HTTP-only cookie stores the `x-brain-key` server-side. The key is never sent to the browser.
- **Two-layer guard**: `middleware.ts` performs a fast cookie-presence check to redirect unauthenticated routes. Each API route handler then calls `requireSession()` for full session validation before processing any request body.
- A separate restricted-content flag (`session.restrictedUnlocked`) gates access to thoughts with elevated `sensitivity_tier`. Unlocking requires a passphrase verified against a SHA-256 hash stored in an environment variable.

The SvelteKit dashboard uses Supabase Auth (anon key + OAuth) via `@supabase/ssr`, with server-side credential injection for MCP calls via private environment variables (`MCP_URL`, `MCP_KEY`).

### Extension Authentication

Extensions (Supabase Edge Functions) use the Supabase service role key with a `DEFAULT_USER_ID` environment variable. The service role key bypasses Row Level Security by default. The `professional-crm` extension is the sole exception: it requires the `rls` primitive for per-user isolation via `auth.uid()`.

### Telegram Webhook Authentication

The Telegram route uses a separate `x-telegram-bot-api-secret-token` header (set when registering the webhook with Telegram). This is independent of the `x-brain-key` used by all other endpoints.

### Locations

- `server/index.ts` — Core MCP server: `x-brain-key` header + `?key=` query param check
- `integrations/kubernetes-deployment/index.ts` — Self-hosted variant: same dual-mode auth
- `recipes/vercel-neon-telegram/src/lib/auth.ts` — `extractKey` + `timingSafeEqual` + three extraction strategies
- `dashboards/open-brain-dashboard-next/lib/auth.ts` — iron-session helpers: `requireSession`, `requireSessionOrRedirect`, `AuthError`
- `dashboards/open-brain-dashboard-next/middleware.ts` — Fast cookie-presence check + redirect
- `dashboards/open-brain-dashboard-next/app/api/restricted/route.ts` — SHA-256 passphrase unlock
- `dashboards/open-brain-dashboard/src/hooks.server.ts` — Supabase Auth bootstrap per request
- `extensions/*/index.ts` — `DEFAULT_USER_ID` + service role key pattern

---

## Error Handling

How errors are handled across the codebase.

### Pattern

Error handling is localized: each module catches errors at the boundary of external calls (LLM APIs, Supabase RPCs, HTTP fetches) and either returns an error response, propagates an exception to the caller, or falls back to a safe default.

Across TypeScript components, the common patterns are:

- **HTTP error propagation**: Check `response.ok` after `fetch`; if false, read the response body and throw an `Error` with the status code and message. Used consistently in Deno Edge Functions and Node.js recipes.
- **JSON parse fallback**: LLM responses that fail to parse (e.g., malformed JSON from `choices[0].message.content`) fall back to a default object (e.g., `{ topics: ["uncategorized"], type: "observation" }`). This prevents a single bad LLM response from crashing the capture pipeline.
- **`Promise.allSettled` for batch operations**: The audit delete route uses `Promise.allSettled` rather than `Promise.all` to allow partial batch success. The response returns counts of `deleted` and `failed` items rather than failing the whole request.
- **`AuthError` for API routes**: The Next.js dashboard defines a custom `AuthError` class thrown by `requireSession()` in API route handlers so they produce proper `401` responses, distinct from `redirect()` used in server components.
- **Startup fail-fast**: Required environment variables are read at module load time with non-null assertions (`Deno.env.get("VAR")!` in Deno, explicit length checks in Next.js auth). Missing config throws immediately at startup rather than silently producing `undefined` at request time.

### Locations

- `server/index.ts` — HTTP error propagation from OpenRouter; JSON parse fallback for LLM metadata
- `integrations/entity-extraction-worker/_shared/helpers.ts` — LLM provider fallback chain; retry on transient errors only; no retry on parse errors
- `integrations/entity-extraction-worker/index.ts` — `ExtractionCostCapError` for runaway container protection; wall-clock budget enforcement (stop at 140s, release stuck items back to `pending`)
- `dashboards/open-brain-dashboard-next/lib/auth.ts` — `AuthError` class; `SESSION_SECRET` length validation at module load
- `dashboards/open-brain-dashboard-next/app/api/audit/delete/route.ts` — `Promise.allSettled` partial batch delete
- `recipes/vercel-neon-telegram/src/lib/auth.ts` — `requireAuth` returns error `Response` object rather than throwing
- `recipes/vercel-neon-telegram/src/app/api/mcp/route.ts` — `try/catch` around MCP handler; returns structured error JSON

---

## Logging

How logging is implemented.

### Pattern

Logging is not centralized. Components use `console.log`, `console.error`, and `console.warn` directly. There is no shared logger wrapper, no log levels enum, and no structured JSON logging format enforced across the codebase.

`console.error` is used for unexpected failures (LLM errors, DB errors, malformed responses). `console.log` is used for operational status messages in scripts and batch-processing workers (e.g., progress counters in import scripts, extraction worker status).

### Configuration

- Logger: Native `console` (Node.js built-in / Deno built-in)
- Levels used: `console.log` (info/progress), `console.error` (errors), `console.warn` (warnings)
- No log aggregation or structured logging configured in any module

### Locations

- `integrations/entity-extraction-worker/index.ts` — Worker progress and error logging
- `integrations/entity-extraction-worker/_shared/helpers.ts` — LLM call errors and provider fallback events
- `recipes/*/` — Script-level progress logging in importer scripts
- `extensions/home-maintenance/index.ts` — Error logging for extension tool calls
- `dashboards/open-brain-dashboard-next/app/api/*/route.ts` — Minimal error logging in catch blocks

---

## Configuration

How configuration is managed.

### Pattern

All components read configuration exclusively from environment variables. There are no config files (JSON, YAML, TOML) with runtime values. Environment variables are read at module load time in most components, which means misconfiguration causes immediate startup failures rather than runtime surprises.

Deno Edge Functions use `Deno.env.get("VAR")!` with non-null assertions. Node.js components use `process.env.VAR`. Both patterns are bare reads with no validation library wrapping the env layer itself (though Zod is used for validating request/response shapes at runtime).

### Sources

- Supabase Secrets: All Deno Edge Function environment variables are deployed as Supabase secrets (`supabase secrets set`)
- Vercel Environment Variables: Node.js dashboard and recipe deployments read from Vercel project env configuration
- Local `.env` files: Single-script import recipes (Python and Node.js) read from local `.env` files via `dotenv` or standard environment injection

### Environment Variables by Component Type

| Component | Key Variables |
|-----------|---------------|
| Core MCP server (`server/`) | `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `OPENROUTER_API_KEY`, `MCP_ACCESS_KEY` |
| Extensions (`extensions/*/`) | `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `DEFAULT_USER_ID`, `MCP_ACCESS_KEY` |
| Entity extraction worker | `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `OPENROUTER_API_KEY` (+ `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` for fallback), `ENTITY_EXTRACTION_MAX_CALLS` |
| Next.js dashboard | `NEXT_PUBLIC_API_URL`, `SESSION_SECRET`, `RESTRICTED_PASSPHRASE_HASH` (optional) |
| SvelteKit dashboard | `PUBLIC_SUPABASE_URL`, `PUBLIC_SUPABASE_ANON_KEY`, `MCP_URL`, `MCP_KEY` |
| Kubernetes deployment | `DATABASE_URL`, `EMBEDDING_API_BASE`, `EMBEDDING_API_KEY`, `CHAT_API_BASE`, `CHAT_API_KEY`, `MCP_ACCESS_KEY` |
| vercel-neon-telegram | `BRAIN_ACCESS_KEY`, `DATABASE_URL`, `OPENAI_API_KEY`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_SECRET_TOKEN` |

### Locations

- `server/index.ts` — Module-load env reads with `!` non-null assertions
- `extensions/*/index.ts` — `Deno.env.get` at module load
- `integrations/entity-extraction-worker/_shared/helpers.ts` — Model name env vars read at call time (`OPENROUTER_EMBEDDING_MODEL`, `OPENROUTER_CLASSIFIER_MODEL`) allowing per-deployment overrides
- `dashboards/open-brain-dashboard-next/lib/auth.ts` — `SESSION_SECRET` length validation at module load
- `recipes/vercel-neon-telegram/src/lib/auth.ts` — `process.env.BRAIN_ACCESS_KEY` read on every call

---

## Data Access

How data is accessed and managed.

### Pattern

Data access is split across two patterns depending on deployment target:

**Supabase RPC pattern (primary):** The canonical server and all extensions use the Supabase JS client (`@supabase/supabase-js`) and call named stored procedures via `supabase.rpc()`. The two core RPCs are:
- `upsert_thought` — content-level deduplication-aware insert
- `match_thoughts` — pgvector cosine similarity search

Table reads/writes outside of RPCs use the Supabase client's fluent query builder (`supabase.from("table").select(...)`, `.insert(...)`, `.update(...)`, `.eq(...)`).

**Raw SQL pattern (Kubernetes / Neon variants):** `integrations/kubernetes-deployment` uses the `postgres` Deno library for direct SQL against a plain PostgreSQL + pgvector instance. `recipes/vercel-neon-telegram` uses the Neon serverless driver with tagged template literals (`sql\`...\``) for most queries and `sql.query(query, params)` for dynamic `WHERE` clauses.

### Two-Step Thought Capture

The core capture flow always uses two separate database operations:
1. Call `upsert_thought` RPC (content dedup + insert)
2. Separately `UPDATE` the row to patch the embedding vector

This split exists because the `upsert_thought` RPC predates vector storage support and does not accept the embedding directly.

### Async Queue Pattern (Entity Extraction)

The entity extraction worker uses a Postgres table (`entity_extraction_queue`) as a work queue. A database trigger (`trg_queue_entity_extraction`) on the `thoughts` table automatically enqueues rows on insert or content change. The worker claims rows atomically by transitioning `status` from `pending` → `processing` via a conditional update. Processed rows become `done` or `skipped`; stuck `processing` rows are released back to `pending` if the worker's wall-clock budget expires.

### ORM / Database Driver

| Component | Database Access Method |
|-----------|----------------------|
| `server/`, all extensions, most integrations | `@supabase/supabase-js` client (fluent builder + `.rpc()`) |
| `integrations/kubernetes-deployment` | `postgres` Deno library (raw SQL, `deno.land/x`) |
| `recipes/vercel-neon-telegram` | `@neondatabase/serverless` driver (tagged template literals) |
| Python import recipes | `requests` HTTP client against Supabase REST API (no ORM) |
| Node.js single-script recipes | Native `fetch` against Supabase REST API |

### Locations

- `server/index.ts` — `supabase.rpc("upsert_thought")`, `supabase.rpc("match_thoughts")`
- `extensions/*/index.ts` — Supabase client fluent queries for domain tables
- `integrations/entity-extraction-worker/index.ts` — Queue drain with atomic status transitions
- `integrations/kubernetes-deployment/index.ts` — Direct SQL with `<=>` pgvector cosine distance
- `recipes/vercel-neon-telegram/src/lib/db.ts` — Neon tagged template literals + dynamic query fallback
- `recipes/*/` — Supabase REST API via `fetch` or Python `requests`

---

## MCP Server Pattern

How Model Context Protocol servers are structured and deployed.

### Pattern

Every MCP server in the codebase follows the same structural pattern:
- A new `McpServer` instance is created **per request**, not once at module load. This is required by the stateless Supabase Edge Function execution model.
- Tools are registered on the server instance via `server.registerTool(name, schema, handler)`.
- Input schemas are defined with `zod` and passed to `registerTool`.
- Transport is `StreamableHTTPTransport` (from `@hono/mcp`) wired into a Hono HTTP framework entry point.
- The entry point is `Deno.serve` for Supabase Edge Functions.

A known Claude Desktop bug requires patching the `Accept: text/event-stream` header into incoming requests before passing to `StreamableHTTPTransport`. This workaround appears in `server/index.ts` and `extensions/family-calendar/index.ts`.

### Locations

- `server/index.ts` — Canonical MCP server pattern
- `extensions/*/index.ts` — Per-extension MCP servers (6 instances)
- `integrations/kubernetes-deployment/index.ts` — Hono-based MCP server (long-lived process, not Edge Function)
- `recipes/vercel-neon-telegram/src/app/api/mcp/route.ts` — `WebStandardStreamableHTTPServerTransport` variant (Next.js App Router)
- `recipes/ob-graph/index.ts` — Standalone MCP server recipe

---

## Parallel Async Operations

How concurrent operations are composed.

### Pattern

Parallel async fan-out is implemented with `Promise.all` wherever two independent operations can proceed concurrently. The primary use case is the thought capture pipeline: embedding generation and LLM metadata extraction are fired in parallel since they depend only on the input text, not on each other.

`Promise.allSettled` is used for batch operations where partial failure is acceptable (e.g., bulk delete in the audit view). It returns results for all items regardless of individual failures, and the caller counts successes and failures separately.

### Locations

- `server/index.ts` — `Promise.all([getEmbedding(text), extractMetadata(text)])` during `capture_thought`
- `integrations/kubernetes-deployment/index.ts` — Same parallel embedding + metadata pattern
- `recipes/vercel-neon-telegram/src/lib/capture.ts` — `Promise.all` for embedding + metadata fan-out
- `dashboards/open-brain-dashboard-next/app/api/audit/delete/route.ts` — `Promise.allSettled` for batch delete

---

## Schema Validation

How runtime input validation is applied.

### Pattern

`zod` is used for runtime schema validation across TypeScript components. MCP tool input schemas are defined as `zod` objects and passed directly to `server.registerTool()`. API route handlers in the vercel-neon-telegram recipe define `zod` schemas for request body parsing.

Python recipes do not use a validation library; they rely on duck typing and manual key existence checks against the parsed JSON/CSV/XML export data.

### Locations

- `server/index.ts` — Zod schemas for all four MCP tool inputs
- `extensions/*/index.ts` — Zod schemas for each extension's MCP tools
- `integrations/kubernetes-deployment/index.ts` — Zod schemas for MCP tool inputs
- `recipes/vercel-neon-telegram/src/lib/types.ts` — Zod schemas for `ThoughtMetadata` and related types
- `recipes/repo-learning-coach/server/index.ts` — Zod schemas for API request validation
- `recipes/repo-learning-coach/server/content-loader.ts` — Zod schema for curriculum frontmatter

---

## LLM Provider Fallback Chain

How LLM provider failures are handled.

### Pattern

The entity extraction worker uses an explicit provider priority list: **OpenRouter → OpenAI → Anthropic**. If the primary provider returns a transient error (network failure, 429 rate limit, 5xx server error), the worker retries with the next provider. Parse errors or bad response shapes do not trigger a retry — those fall through immediately to the next provider.

Individual recipes that call LLMs directly (entity-wiki, thought-enrichment, typed-edge-classifier, wiki-synthesis) call Anthropic directly without a fallback chain.

The core MCP server uses OpenRouter exclusively for both embeddings (`text-embedding-3-small`) and metadata extraction (`gpt-4o-mini`).

### Timeout Enforcement

Every outbound LLM call in the entity extraction worker is wrapped in an `AbortController` with a 60-second timeout (`FETCH_TIMEOUT_MS`). Without this, a stalled upstream LLM provider can consume the entire Edge Function wall-clock budget (150s) on a single call.

### Locations

- `integrations/entity-extraction-worker/_shared/helpers.ts` — Multi-provider fallback chain with `AbortController` timeout per call
- `server/index.ts` — OpenRouter-only (no fallback)
- `recipes/*/` — Anthropic direct calls (no fallback)

---

## Prompt Injection Defense

How user-supplied content is protected from LLM prompt injection.

### Pattern

In the entity extraction worker, thought content is wrapped in `<thought_content>` XML-like delimiters before insertion into LLM prompts. Any literal occurrences of those delimiter tags within the thought content are escaped before insertion. The system prompt explicitly instructs the LLM to return empty arrays if it detects an injection attempt.

### Locations

- `integrations/entity-extraction-worker/_shared/helpers.ts` — XML delimiter wrapping and tag escaping

---

## Sensitivity and Content Gating

How sensitive content is classified and access-controlled.

### Pattern

A `sensitivity_tier` column on the `thoughts` table has three levels: `standard`, `personal`, `restricted`. Escalation is one-way: once a thought is classified at a higher tier, it cannot be downgraded by pipeline code. Unrecognized tier values default to `"personal"` (erring toward privacy).

The entity extraction worker skips thoughts with `metadata->>'generated_by'` set (system-generated artifacts), preventing circular entity extraction from AI-authored content.

The Next.js dashboard exposes a second-factor unlock flow: restricted thoughts are hidden by default even after login. Unlocking requires a passphrase verified against a SHA-256 env var hash.

### Locations

- `integrations/entity-extraction-worker/_shared/helpers.ts` — `resolveSensitivityTier`, escalation-only logic
- `schemas/enhanced-thoughts/schema.sql` — `sensitivity_tier` column definition
- `dashboards/open-brain-dashboard-next/app/api/restricted/route.ts` — SHA-256 passphrase unlock
- `dashboards/open-brain-dashboard-next/lib/api.ts` — `exclude_restricted` parameter on all data-fetch calls

---

## Shared Code Distribution

How shared utilities are distributed across independently deployed functions.

### Pattern

Supabase Edge Functions cannot share code across function directories at deploy time. Shared utility code (`_shared/config.ts`, `_shared/helpers.ts`) is copied by hand into each Edge Function directory that needs it. This is a copy-not-symlink pattern enforced by the Supabase deployment constraint.

### Locations

- `integrations/entity-extraction-worker/_shared/` — Shared extraction pipeline utilities duplicated per Edge Function

---

## CI / Contribution Governance

How pull request quality is enforced automatically.

### Pattern

A two-stage GitHub Actions pipeline governs all contributions:

1. **`ob1-gate.yml`** — Deterministic shell-script checks run on every PR. Validates `metadata.json` schema, file structure, absence of credentials, remote MCP pattern compliance, and safety rules. Uploads a structured artifact (`ob1-pr-gate-context`). Has `contents: read` only; no write permissions.
2. **`ob1-pr-followups.yml`** — Triggers on gate completion. Downloads the artifact, posts an idempotent PR comment (identified by an HTML marker), and conditionally invokes `anthropics/claude-code-action` for qualitative review. Only fires for trusted contributors (OWNER, MEMBER, COLLABORATOR, CONTRIBUTOR) with fewer than 3 open PRs.

The two stages are decoupled so the gate can fail fast without consuming LLM API credits.

PR comments are idempotent: the follow-up workflow searches for an existing bot comment by HTML marker and updates it rather than posting a duplicate on each push.

### Locations

- `.github/workflows/ob1-gate.yml` — Deterministic structural gate
- `.github/workflows/ob1-pr-followups.yml` — Artifact handoff, PR comment, conditional LLM review
- `.github/workflows/claude-review.yml` — Manual-dispatch qualitative review
- `.github/metadata.schema.json` — JSON schema for `metadata.json` validation
