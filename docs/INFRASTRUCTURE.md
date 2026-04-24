# INFRASTRUCTURE.md

> Operations and infrastructure facts for this codebase.

## CI/CD

### Platform

GitHub Actions. All workflows live in `.github/workflows/`.

### Workflows

| Workflow | File | Trigger | Purpose |
|----------|------|---------|---------|
| OB1 PR Gate | `ob1-gate.yml` | PR opened/sync/reopened → `main` | Deterministic structural and safety checks; uploads gate artifact |
| OB1 PR Follow-Ups | `ob1-pr-followups.yml` | `workflow_run` completion of OB1 PR Gate | Posts idempotent PR comment; conditionally invokes Claude auto-review |
| Claude PR Review | `claude-review.yml` | `workflow_dispatch` (manual) | LLM-backed qualitative review via `anthropics/claude-code-action@v1` |
| Claude Issue Triage | `claude-issue-triage.yml` | Issues opened/reopened | Classifies, checks for duplicates, posts triage comment |
| Auto-Label PRs | `auto-label.yml` | `pull_request_target` opened/sync → `main` | Applies category labels based on changed file paths |
| Markdown Lint | `markdown-lint.yml` | PR → `main` (paths: `**/*.md`) | Runs `markdownlint-cli2` on changed Markdown files |
| Discord Release Announcement | `discord-announce.yml` | Release published | Posts release embed to Discord via webhook |
| Release Drafter | `release-drafter.yml` | Push to `main` | Maintains draft release changelog |
| Welcome New Contributors | `welcome-new-contributors.yml` | Issue opened; `pull_request_target` opened | Posts welcome comment to first-time contributors |

### Two-Stage PR Review Pipeline

Stage 1 — `ob1-gate.yml`: Runs a shell-script battery of deterministic checks, writes results to a structured artifact (`ob1-pr-gate-context`) retained for 7 days, and fails the workflow if any check fails.

Stage 2 — `ob1-pr-followups.yml`: Downloads the gate artifact after completion, posts (or updates) an idempotent review comment on the PR, manages `security-blocked` and `needs-maintainer-triage` labels, and — for trusted contributors whose PR passed the gate — invokes `anthropics/claude-code-action@v1` for qualitative review.

### PR Gate Checks

| Rule | What It Checks |
|------|---------------|
| Folder structure | All changed files are within allowed top-level directories |
| Required files | Every contribution folder has `README.md` and `metadata.json` |
| Metadata valid | `metadata.json` is valid JSON and passes `.github/metadata.schema.json` |
| No credentials | Scans for API key patterns; blocks `.env` files with real values |
| SQL safety | No `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`, unguarded `DELETE FROM`, or `ALTER TABLE thoughts DROP/ALTER COLUMN` |
| Category artifacts | Category-specific file requirements (see Contribution Governance section) |
| PR format | Title matches `[category] Description` pattern |
| No binary blobs | No files over 1 MB; no `.exe`, `.dmg`, `.zip`, `.tar.gz`, etc. |
| README completeness | READMEs include prerequisites, numbered steps, and expected outcome |
| Contribution dependencies | Declared `requires_primitives` and `requires_skills` exist and are linked in README |
| Remote MCP pattern | No `claude_desktop_config.json`, `StdioServerTransport`, or local `mcpServers` patterns |
| Scope check | Changes confined to the contribution's own folder |
| Internal links | Relative links in READMEs resolve to existing files |
| Tool audit link | Extensions and integrations link to `docs/05-tool-audit.md` |

### GitHub Actions Runtime Details

- Runner: `ubuntu-latest` (all workflows)
- Node.js version for markdown linting: `22`
- Python metadata validator: `check-jsonschema` (installed via `pip` per run)
- Claude model for automated reviews and triage: `claude-haiku` (via `anthropics/claude-code-action@v1`)
- Claude PR Review timeout: `20` minutes

### Required CI Secrets

| Secret | Used By |
|--------|---------|
| `GITHUB_TOKEN` | All workflows (GitHub-provided) |
| `ANTHROPIC_API_KEY` | `ob1-pr-followups.yml`, `claude-review.yml`, `claude-issue-triage.yml` |
| `DISCORD_WEBHOOK_URL` | `discord-announce.yml` |

---

## Containerization

### Docker

A Dockerfile exists for the self-hosted Kubernetes integration only (`integrations/kubernetes-deployment/Dockerfile`). It is not part of the primary Supabase-hosted deployment path.

| Attribute | Value |
|-----------|-------|
| Base image | `denoland/deno:2.3.3` |
| Working directory | `/app` |
| Exposed port | `8000` |
| Runtime user | `deno` (non-root) |
| Deno permissions | `--allow-net`, `--allow-env`, `--allow-read` |
| Entry point | `deno run ... index.ts` |

```bash
# Build
docker build -t openbrain-mcp-server integrations/kubernetes-deployment/

# Run (all env vars are required)
docker run -p 8000:8000 \
  -e DB_HOST=127.0.0.1 \
  -e DB_PORT=5432 \
  -e DB_NAME=openbrain \
  -e DB_USER=postgres \
  -e DB_PASSWORD=<password> \
  -e MCP_ACCESS_KEY=<key> \
  -e EMBEDDING_API_BASE=https://openrouter.ai/api/v1 \
  -e EMBEDDING_API_KEY=<key> \
  -e EMBEDDING_MODEL=openai/text-embedding-3-small \
  -e CHAT_API_BASE=https://openrouter.ai/api/v1 \
  -e CHAT_API_KEY=<key> \
  -e CHAT_MODEL=openai/gpt-4o-mini \
  openbrain-mcp-server
```

There is no `docker-compose.yml` in this repository.

---

## Kubernetes (Self-Hosted Deployment)

A Kubernetes manifest is provided as a community-supported integration for self-managed infrastructure. This is an explicitly documented exception to the repo's remote-MCP-only rule.

- File: `integrations/kubernetes-deployment/k8s/openbrain.yml`
- Namespace: `openbrain`
- Workload type: `StatefulSet` (1 replica)

### Pod Layout

The StatefulSet runs a single multi-container pod. Containers share the network namespace and communicate over `127.0.0.1`.

| Container | Image | Purpose | Port |
|-----------|-------|---------|------|
| `db` | `ankane/pgvector:v0.5.1` | PostgreSQL with pgvector extension | 5432 |
| `mcp-server` | `openbrain-mcp-server:latest` | Deno MCP server (Hono HTTP) | 8000 |

### Resource Limits

| Container | CPU Request | Memory Request | Memory Limit |
|-----------|-------------|---------------|-------------|
| `db` | 250m | 256Mi | 1Gi |
| `mcp-server` | 100m | 128Mi | 512Mi |

### Storage

- PostgreSQL data: `hostPath` volume at `/var/openbrain/db` on the host node
- Init SQL: mounted from a `ConfigMap` (`openbrain-init-sql`); creates the `vector` extension, the `thoughts` table, indexes, and the `match_thoughts` PL/pgSQL function

### Service

- Type: `ClusterIP`
- Port: `8000` (MCP HTTP)

An ingress block for HTTPS external access is included but commented out. Operators must configure their own ingress controller and TLS.

### Kubernetes Secrets Required

All sensitive values are read from a `Secret` named `openbrain-secret`:

| Key | Purpose |
|-----|---------|
| `postgres-password` | PostgreSQL superuser password |
| `mcp-access-key` | `x-brain-key` for MCP authentication |
| `embedding-api-key` | Embedding API key |
| `chat-api-key` | Chat completion API key |

---

## Configuration Management

### Approach

All credentials and environment-specific values are passed via environment variables. No credentials are committed to the repository. The CI gate scans every pull request for credential patterns and blocks merges on detection.

### .env Files

Each deployable component ships a `.env.example` file. Operators copy to `.env` and fill in values. The following components have `.env.example` files:

- `dashboards/open-brain-dashboard-next/.env.example`
- `dashboards/open-brain-dashboard/.env.example`
- All six `extensions/*/` directories
- Multiple `recipes/*/` directories

### Environment Variables by Component

#### Core Extensions and Server (Supabase Edge Functions)

| Variable | Purpose | Set By |
|----------|---------|--------|
| `SUPABASE_URL` | Supabase project URL | Supabase automatically |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key | Supabase automatically |
| `MCP_ACCESS_KEY` | `x-brain-key` for MCP authentication | Operator manually |

#### Next.js Dashboard (`dashboards/open-brain-dashboard-next`)

| Variable | Purpose | Required |
|----------|---------|---------|
| `NEXT_PUBLIC_API_URL` | URL of the Open Brain REST API Edge Function | Yes |
| `SESSION_SECRET` | 32+ char secret for iron-session cookie encryption | Yes |
| `RESTRICTED_PASSPHRASE_HASH` | SHA-256 hash for unlocking sensitive thoughts | No |

#### Vercel + Neon + Telegram Recipe

| Variable | Purpose | Required |
|----------|---------|---------|
| `DATABASE_URL` | Neon Postgres connection string | Yes |
| `OPENAI_API_KEY` | OpenAI embeddings key | Yes |
| `BRAIN_ACCESS_KEY` | MCP/API authentication key | Yes |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token | No |
| `TELEGRAM_WEBHOOK_SECRET` | Telegram webhook HMAC secret | No |
| `APP_URL` | Deployed Vercel URL | No (set post-deploy) |

#### Kubernetes Deployment

| Variable | Purpose |
|----------|---------|
| `DB_HOST` | PostgreSQL host (typically `127.0.0.1` inside pod) |
| `DB_PORT` | PostgreSQL port |
| `DB_NAME` | Database name |
| `DB_USER` | Database user |
| `DB_PASSWORD` | Database password |
| `MCP_ACCESS_KEY` | `x-brain-key` for MCP authentication |
| `EMBEDDING_API_BASE` | Embedding API base URL |
| `EMBEDDING_API_KEY` | Embedding API key |
| `EMBEDDING_MODEL` | Embedding model name (e.g., `openai/text-embedding-3-small`) |
| `CHAT_API_BASE` | Chat completion API base URL |
| `CHAT_API_KEY` | Chat completion API key |
| `CHAT_MODEL` | Chat model name (e.g., `openai/gpt-4o-mini`) |
| `PORT` | HTTP server port (default: `8000`) |

---

## Deployment

### Primary Method: Supabase Edge Functions (Deno)

The canonical deployment target for all MCP servers is Supabase Edge Functions. Each function is deployed independently. There is no unified build artifact or mono-deploy step.

```bash
supabase functions deploy <function-name>
```

### Deployment Units

| Unit | Runtime | Target |
|------|---------|--------|
| Core MCP server (`server/`) | Deno Edge Function | Supabase Functions |
| Extension MCP servers ×6 (`extensions/*/index.ts`) | Deno Edge Function | Supabase Functions (one deploy per extension) |
| Entity extraction worker (`integrations/entity-extraction-worker/`) | Deno Edge Function, cron-invoked | Supabase Functions |
| Next.js dashboard (`dashboards/open-brain-dashboard-next/`) | Node.js 18+ | Vercel (or any Next.js host) |
| SvelteKit dashboard (`dashboards/open-brain-dashboard/`) | Node.js 22.x | Vercel (adapter-vercel) |
| Vercel + Neon + Telegram (`recipes/vercel-neon-telegram/`) | Node.js / Next.js App Router | Vercel |
| Repo learning coach (`recipes/repo-learning-coach/`) | Node.js (Express + Vite) | Local / self-hosted |
| Kubernetes MCP server (`integrations/kubernetes-deployment/`) | Deno long-lived process | Kubernetes StatefulSet |
| Supabase database | PostgreSQL + pgvector | Supabase managed service |
| CI workflows (`.github/workflows/`) | GitHub Actions | GitHub |

### Schema Migration Apply Order

Schema extensions must be applied in strict dependency order. Applying out of order raises a runtime exception with a descriptive message.

```
docs/01-getting-started.md Step 2.6   (adds thoughts.content_fingerprint — prerequisite)
    ↓
schemas/enhanced-thoughts/schema.sql   (classification columns, full-text RPCs)
    ↓
schemas/entity-extraction/schema.sql   (knowledge graph tables, extraction queue trigger)
    ↓
schemas/typed-reasoning-edges/schema.sql  (typed thought_edges; adds temporal columns to edges)
```

### Authentication at Runtime

| Mechanism | Used By | Notes |
|-----------|---------|-------|
| `x-brain-key` header | server/, extensions/, kubernetes-deployment | Primary API client pattern |
| `?key=` query parameter | server/, extensions/, kubernetes-deployment | Required for Claude Desktop and ChatGPT (cannot send custom headers) |
| `Authorization: Bearer` | vercel-neon-telegram | Same key, different format |
| iron-session encrypted cookie | open-brain-dashboard-next | Stores `x-brain-key` server-side; never exposed to browser |
| Supabase Auth (anon + OAuth) | open-brain-dashboard (SvelteKit) | Managed via `@supabase/ssr` |
| `x-telegram-bot-api-secret-token` | vercel-neon-telegram /api/telegram | Separate from brain key |

---

## Markdown Linting

- Tool: `markdownlint-cli2` (Node.js `22`)
- Config: `.github/.markdownlint.jsonc`
- Trigger: PR opening/sync on any `**/*.md` file targeting `main`
- Disabled rules: `MD013` (line length), `MD024` (duplicate headings), `MD033` (inline HTML), `MD034` (bare URLs), `MD025`, `MD026`, `MD032`, `MD036`, `MD040`, `MD041`, `MD060`

---

## Contribution Governance Infrastructure

### Contribution Category Requirements

| Category | Additional Required Artifacts |
|----------|------------------------------|
| `recipes` | Code files (`.sql/.ts/.js/.py`) or 3+ numbered README steps |
| `schemas` | At least one `.sql` file |
| `dashboards` | Frontend code (`.html/.jsx/.tsx/.vue/.svelte`) or `package.json` |
| `integrations` | Code files (`.ts/.js/.py`) |
| `skills` | A skill file (`SKILL.md`, `*.skill.md`, or `*-skill.md`) |
| `primitives` | README with 200+ words |
| `extensions` | At least one `.sql` file and code files (`.ts/.js/.py`); must also link to `docs/05-tool-audit.md` |

### Contributor Trust Policy

| Author Association | Eligible for Claude Auto-Review |
|-------------------|--------------------------------|
| OWNER, MEMBER, COLLABORATOR, CONTRIBUTOR | Yes (after gate passes, not draft, not over quota) |
| FIRST_TIME_CONTRIBUTOR, NONE | No (gate runs; human review required) |

Contributors with more than 3 open PRs receive the `needs-maintainer-triage` label.

### Metadata Schema

All contributions must include a `metadata.json` validated against `.github/metadata.schema.json` (JSON Schema Draft 2020-12).

Required fields: `name`, `description`, `category`, `author` (with `name`), `version` (semver), `requires` (with `open_brain: true`), `tags`, `difficulty` (`beginner`/`intermediate`/`advanced`), `estimated_time`.

Optional fields: `requires_primitives`, `requires_skills`, `learning_order` (extensions only), `created`, `updated`.

---

## External Service Dependencies at Runtime

| Service | Purpose | Consuming Components |
|---------|---------|---------------------|
| Supabase (hosted) | PostgreSQL + pgvector database, Edge Function hosting, auth | server/, all extensions, entity-extraction-worker, both dashboards, most recipes |
| OpenRouter API | Embeddings (`text-embedding-3-small`) and LLM metadata extraction (`gpt-4o-mini`) | server/, entity-extraction-worker, repo-learning-coach brain bridge |
| Anthropic Claude API | Entity extraction, classification, enrichment, synthesis | entity-extraction-worker, several recipes and skills |
| OpenAI API | Embeddings via Vercel AI SDK | vercel-neon-telegram |
| Neon Postgres | Alternative Postgres backend with pgvector | vercel-neon-telegram |
| Telegram Bot API | Thought capture and retrieval over Telegram | vercel-neon-telegram |
| Gmail API / Google OAuth2 | Email history import | recipes/email-history-import |
| Ollama (local) | Locally-run embedding generation | recipes/local-ollama-embeddings |
| Vercel | Frontend hosting | open-brain-dashboard (SvelteKit), vercel-neon-telegram |
| GitHub Actions | CI/CD workflow execution | .github/workflows/ |
| `anthropics/claude-code-action@v1` | LLM-backed PR review and issue triage | ob1-pr-followups.yml, claude-review.yml, claude-issue-triage.yml |
