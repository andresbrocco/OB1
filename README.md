# Open Brain (OB1)

> A persistent AI memory system -- one database, one MCP protocol, any AI client. Capture thoughts from Claude, ChatGPT, Cursor, or any MCP-compatible tool. Search them semantically. Build on top of a single Supabase + pgvector database that every component reads from and writes to.

## Quick Start

### Prerequisites

- A [Supabase](https://supabase.com) account (free tier)
- An [OpenRouter](https://openrouter.ai) API key (~$5 in credits, lasts months)
- [Supabase CLI](https://supabase.com/docs/guides/local-development/cli/getting-started) installed
- An MCP-compatible AI client (Claude Desktop, ChatGPT, Claude Code, Cursor, etc.)

### Setup

The full guided setup takes about 30 minutes and requires zero coding experience.

```bash
# 1. Set up your Supabase project and database tables
#    Follow the step-by-step guide:
#    docs/01-getting-started.md (Steps 1-5)

# 2. Create and deploy the MCP server
supabase functions new open-brain-mcp
curl -o supabase/functions/open-brain-mcp/index.ts \
  https://raw.githubusercontent.com/NateBJones-Projects/OB1/main/server/index.ts
curl -o supabase/functions/open-brain-mcp/deno.json \
  https://raw.githubusercontent.com/NateBJones-Projects/OB1/main/server/deno.json

# 3. Set secrets
supabase secrets set MCP_ACCESS_KEY=<your-access-key>
supabase secrets set OPENROUTER_API_KEY=<your-openrouter-key>

# 4. Deploy
supabase functions deploy open-brain-mcp --no-verify-jwt
```

### Connect Your AI

Your MCP server URL will be:

```
https://<PROJECT_REF>.supabase.co/functions/v1/open-brain-mcp
```

- **Claude Desktop:** Settings > Connectors > Add custom connector > paste URL with `?key=<access-key>`
- **ChatGPT:** Settings > Apps & Connectors > Create > paste URL with `?key=<access-key>`
- **Claude Code:** `claude mcp add --transport http open-brain <URL> --header "x-brain-key: <access-key>"`

See [docs/01-getting-started.md](docs/01-getting-started.md) for the complete walkthrough with OS-specific instructions.

### Verify Setup

```bash
# Check the server is active
supabase functions list
# Should show open-brain-mcp as ACTIVE

# In your AI client, try:
# "Remember this: testing my Open Brain setup"
# Then: "What did I capture about testing?"
```

## MCP Tools

| Tool | Description | Key Inputs |
|------|-------------|------------|
| `capture_thought` | Save a new thought with auto-generated embedding and metadata | `content` (string) |
| `search_thoughts` | Semantic vector search over the thoughts table | `query`, `limit`, `threshold` |
| `list_thoughts` | List recent thoughts with optional filters | `limit`, `type`, `topic`, `person`, `days` |
| `thought_stats` | Aggregate statistics: totals, type breakdown, top topics and people | -- |

## Port Reference

| Port | Service | Description |
|------|---------|-------------|
| 8000 | open-brain-mcp | Main MCP server (Supabase Edge Function / Deno runtime) |
| 5432 | postgres (pgvector) | PostgreSQL with pgvector (self-hosted Kubernetes deployments only) |
| 8787 | repo-learning-coach | Express dev server for the Repo Learning Coach recipe |

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Supabase project URL | Yes (auto-provided in Edge Functions) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase secret key for service-role access | Yes (auto-provided in Edge Functions) |
| `MCP_ACCESS_KEY` | Shared access key for authenticating MCP requests (`x-brain-key` header) | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key for embeddings and LLM metadata extraction | Yes |

Individual recipes and dashboards may require additional variables. Check their `.env.example` files for specifics.

## Common Commands

### Deploy

```bash
# Deploy core MCP server
supabase functions deploy open-brain-mcp --no-verify-jwt

# Set Supabase secrets
supabase secrets set MCP_ACCESS_KEY=<key>
supabase secrets set OPENROUTER_API_KEY=<key>

# Deploy to self-hosted Kubernetes
kubectl apply -f k8s/secrets.yml -f k8s/openbrain.yml
```

### Dashboard Development

```bash
# Next.js dashboard
cd dashboards/open-brain-dashboard-next
cp .env.example .env.local
npm install
npm run dev

# SvelteKit dashboard
cd dashboards/open-brain-dashboard
cp .env.example .env
npm install
npm run dev
```

### Testing

```bash
# Only recipes/vercel-neon-telegram has a test suite currently
cd recipes/vercel-neon-telegram
npm test
```

## Project Structure

```
OB1/
├── server/                  # Core MCP server (Supabase Edge Function)
├── schemas/                 # Additive database migrations
│   ├── enhanced-thoughts/   # Additional thought metadata columns
│   ├── entity-extraction/   # Knowledge graph entity tables
│   ├── typed-reasoning-edges/ # Typed relationship edges
│   └── workflow-status/     # Workflow tracking schema
├── extensions/              # Curated domain-specific MCP servers (learning path)
│   ├── family-calendar/     # Family calendar management
│   ├── home-maintenance/    # Home maintenance tracking
│   ├── household-knowledge/ # Household knowledge base
│   ├── job-hunt/            # Job search tracking
│   ├── meal-planning/       # Meal planning and shopping lists
│   └── professional-crm/   # Professional contact management
├── dashboards/              # Browser UIs
│   ├── open-brain-dashboard-next/  # Next.js 14 dashboard
│   └── open-brain-dashboard/       # SvelteKit 5 dashboard
├── integrations/            # Background workers and deployment variants
│   ├── entity-extraction-worker/   # Cron-driven knowledge graph builder
│   └── kubernetes-deployment/      # Self-hosted K8s deployment
├── recipes/                 # Standalone capability builds
│   ├── chatgpt-conversation-import/
│   ├── google-activity-import/
│   ├── instagram-import/
│   ├── x-twitter-import/
│   ├── grok-export-import/
│   ├── journals-blogger-import/
│   ├── ob-graph/
│   ├── repo-learning-coach/
│   ├── vercel-neon-telegram/
│   └── ...
├── skills/                  # AI behavioral prompt packs
├── primitives/              # Reusable concept guides
├── docs/                    # Setup guides, FAQ, companion prompts
└── resources/               # Official companion files
```

### Schema Migration Order

Schemas must be applied in this order:

```
docs/01-getting-started.md Step 2.6 (adds thoughts.content_fingerprint)
    |
schemas/enhanced-thoughts/schema.sql
    |
schemas/entity-extraction/schema.sql
    |
schemas/typed-reasoning-edges/schema.sql
```

## Module READMEs

### Core

- [server/](server/README.md) -- Core MCP server: thought capture, semantic search, stats

### Dashboards

- [dashboards/open-brain-dashboard-next/](dashboards/open-brain-dashboard-next/README.md) -- Next.js 14 browser UI
- [dashboards/open-brain-dashboard/](dashboards/open-brain-dashboard/README.md) -- SvelteKit 5 browser UI

### Extensions (Curated Learning Path)

- [extensions/household-knowledge/](extensions/household-knowledge/README.md) -- Household knowledge base
- [extensions/family-calendar/](extensions/family-calendar/README.md) -- Family calendar management
- [extensions/home-maintenance/](extensions/home-maintenance/README.md) -- Home maintenance tracking
- [extensions/meal-planning/](extensions/meal-planning/README.md) -- Meal planning and shopping lists
- [extensions/professional-crm/](extensions/professional-crm/README.md) -- Professional CRM
- [extensions/job-hunt/](extensions/job-hunt/README.md) -- Job search tracking

### Schemas

- [schemas/enhanced-thoughts/](schemas/enhanced-thoughts/README.md) -- Additional thought metadata
- [schemas/entity-extraction/](schemas/entity-extraction/README.md) -- Knowledge graph entities
- [schemas/typed-reasoning-edges/](schemas/typed-reasoning-edges/README.md) -- Typed relationship edges
- [schemas/workflow-status/](schemas/workflow-status/README.md) -- Workflow tracking

### Integrations

- [integrations/entity-extraction-worker/](integrations/entity-extraction-worker/README.md) -- Cron-driven knowledge graph builder
- [integrations/kubernetes-deployment/](integrations/kubernetes-deployment/README.md) -- Self-hosted K8s deployment

### Recipes

- [recipes/chatgpt-conversation-import/](recipes/chatgpt-conversation-import/README.md) -- Import ChatGPT data export
- [recipes/google-activity-import/](recipes/google-activity-import/README.md) -- Import Google activity data
- [recipes/instagram-import/](recipes/instagram-import/README.md) -- Import Instagram data
- [recipes/x-twitter-import/](recipes/x-twitter-import/README.md) -- Import X/Twitter data
- [recipes/grok-export-import/](recipes/grok-export-import/README.md) -- Import Grok conversations
- [recipes/journals-blogger-import/](recipes/journals-blogger-import/README.md) -- Import Blogger journals
- [recipes/fingerprint-dedup-backfill/](recipes/fingerprint-dedup-backfill/README.md) -- Backfill dedup fingerprints
- [recipes/local-ollama-embeddings/](recipes/local-ollama-embeddings/README.md) -- Local embeddings via Ollama
- [recipes/ob-graph/](recipes/ob-graph/README.md) -- Knowledge graph generator
- [recipes/repo-learning-coach/](recipes/repo-learning-coach/README.md) -- Interactive repo learning tool
- [recipes/vercel-neon-telegram/](recipes/vercel-neon-telegram/README.md) -- Alternative Vercel + Neon + Telegram stack

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| 401 Unauthorized | Access key mismatch | Verify `MCP_ACCESS_KEY` secret matches the key in your connection URL |
| "Permission denied for table thoughts" | Missing GRANT on newer Supabase projects | Run `GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE public.thoughts TO service_role;` |
| Search returns no results | No thoughts captured yet, or threshold too high | Capture a test thought first; try `threshold: 0.3` for wider matching |
| Tools don't appear in Claude Desktop | Connector not enabled for conversation | Click "+" > Connectors > toggle Open Brain on |
| Slow first response | Edge Function cold start | Normal; subsequent calls are faster |
| ChatGPT ignores MCP tools | Developer Mode not enabled | Settings > Apps & Connectors > Advanced settings > toggle Developer mode ON |

See [docs/01-getting-started.md](docs/01-getting-started.md) for detailed troubleshooting with step-by-step solutions.

## Documentation

- [CONTEXT.md](CONTEXT.md) -- Architecture overview (AI agents)
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) -- System design, components, and data flows
- [docs/DEPENDENCIES.md](docs/DEPENDENCIES.md) -- Module relationships and dependency graph
- [docs/PATTERNS.md](docs/PATTERNS.md) -- Auth, error handling, MCP server structure
- [docs/TESTING.md](docs/TESTING.md) -- Test organization and commands
- [docs/SECURITY.md](docs/SECURITY.md) -- Authentication, secrets management
- [docs/INFRASTRUCTURE.md](docs/INFRASTRUCTURE.md) -- CI/CD, Docker, Kubernetes deployment
- [docs/01-getting-started.md](docs/01-getting-started.md) -- Full setup guide (start here)

## Contributing

Every contribution lives in its own subfolder under the appropriate category and must include `README.md` + `metadata.json`. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full guide, including:

- Where your contribution belongs (`recipes/`, `schemas/`, `dashboards/`, `integrations/`, `skills/`)
- Required README sections and formatting conventions
- The `metadata.json` template and schema
- PR title format: `[category] Short description`
- Branch convention: `contrib/<github-username>/<short-description>`
- Automated review rules that must pass before human review

**Guard rails:** Never modify the core `thoughts` table structure. No credentials in files. No binary blobs over 1MB. No destructive SQL (`DROP TABLE`, `TRUNCATE`, unqualified `DELETE FROM`). MCP servers must be remote (Supabase Edge Functions), not local.

## License

[FSL-1.1-MIT](LICENSE.md) -- Functional Source License. No commercial derivative works. Converts to MIT after two years.
