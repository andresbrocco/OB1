# server

> Core MCP server implementing the Open Brain memory protocol as a Hono HTTP application, deployable as a Supabase Edge Function or via Kubernetes.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | URL of your Supabase project (e.g. `https://<ref>.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key — bypasses RLS for server-side writes | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key — used for embeddings (`text-embedding-3-small`) and metadata extraction (`gpt-4o-mini`) | Yes |
| `MCP_ACCESS_KEY` | Secret key clients must supply via `x-brain-key` header or `?key=` query param | Yes |

### HTTP Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `OPTIONS` | `*` | CORS preflight — required for Claude Desktop and browser-based clients |
| `POST` / `GET` / `DELETE` | `*` | MCP protocol endpoint — all MCP tool calls route here. Auth via `x-brain-key` header or `?key=` param. |

Authentication is enforced on every non-OPTIONS request. A missing or incorrect key returns `401 {"error": "Invalid or missing access key"}`.

```bash
# Verify the server is reachable and your key is valid
curl -X POST https://<project-ref>.supabase.co/functions/v1/open-brain-mcp \
  -H "x-brain-key: YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'

# Alternatively, pass the key as a query param
curl -X POST "https://<project-ref>.supabase.co/functions/v1/open-brain-mcp?key=YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

### Port

| Port | Service | Notes |
|------|---------|-------|
| `8000` | open-brain-mcp | `Deno.serve()` default — used in local dev and Kubernetes deployments |

### MCP Tools

| Tool | Description |
|------|-------------|
| `search_thoughts` | Semantic vector search over `thoughts` table using pgvector's `match_thoughts` RPC |
| `list_thoughts` | List recent thoughts; filter by `type`, `topic`, `person`, or `days` |
| `thought_stats` | Aggregate statistics: total count, type breakdown, top topics, people mentioned |
| `capture_thought` | Save a new thought — auto-generates embedding via OpenRouter and extracts structured metadata |

### Commands

```bash
# Deploy to Supabase Edge Functions (production)
supabase functions deploy open-brain-mcp --no-verify-jwt

# Run locally with Deno (development)
deno run --allow-net --allow-env --allow-read index.ts
```

### Configuration

| File | Purpose |
|------|---------|
| `deno.json` | Deno import map — pins all npm dependencies (`hono`, `@hono/mcp`, `@modelcontextprotocol/sdk`, `zod`, `@supabase/supabase-js`) |

### Database Tables

| Table | Usage |
|-------|-------|
| `thoughts` | Primary store — reads `content`, `metadata`, `created_at`, `embedding` columns |

**Database RPCs used:**

| Function | Called by |
|----------|-----------|
| `match_thoughts` | `search_thoughts` — pgvector cosine similarity search |
| `upsert_thought` | `capture_thought` — content-dedup insert/update |

### Prerequisites

- Deno 1.40+ (for local dev)
- Supabase project with `pgvector` enabled and the `thoughts` table + `match_thoughts` / `upsert_thought` RPCs deployed
- OpenRouter account with access to `openai/text-embedding-3-small` and `openai/gpt-4o-mini`
- Supabase CLI (`supabase` v1.x+) for deployment

## Common Tasks

### Deploy to Supabase Edge Functions

```bash
# From the repo root — Supabase CLI picks up /server/index.ts as the function body
supabase functions deploy open-brain-mcp --no-verify-jwt
```

The `--no-verify-jwt` flag is required because authentication is handled internally via `MCP_ACCESS_KEY`, not Supabase JWTs.

### Set Environment Variables on Supabase

```bash
supabase secrets set SUPABASE_URL=https://<ref>.supabase.co
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=<service-role-key>
supabase secrets set OPENROUTER_API_KEY=<openrouter-key>
supabase secrets set MCP_ACCESS_KEY=<your-chosen-secret>
```

### Connect Claude Desktop

In Claude Desktop: Settings → Connectors → Add custom connector → paste the deployed function URL with your key:

```
https://<project-ref>.supabase.co/functions/v1/open-brain-mcp?key=YOUR_MCP_ACCESS_KEY
```

### Run Locally for Development

```bash
cd server
SUPABASE_URL=... SUPABASE_SERVICE_ROLE_KEY=... OPENROUTER_API_KEY=... MCP_ACCESS_KEY=... \
  deno run --allow-net --allow-env --allow-read index.ts
```

Server listens on `http://localhost:8000`.

### Capture a Thought via MCP

```bash
curl -X POST http://localhost:8000 \
  -H "x-brain-key: YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "capture_thought",
      "arguments": { "content": "Decided to use pgvector for semantic search instead of a dedicated vector DB." }
    }
  }'
```

### Search Thoughts via MCP

```bash
curl -X POST http://localhost:8000 \
  -H "x-brain-key: YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "search_thoughts",
      "arguments": { "query": "vector database decisions", "limit": 5, "threshold": 0.6 }
    }
  }'
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `401 Invalid or missing access key` | `x-brain-key` header or `?key=` param is missing or wrong | Check `MCP_ACCESS_KEY` secret matches what the client sends |
| `OpenRouter embeddings failed: 401` | `OPENROUTER_API_KEY` is invalid or expired | Rotate the key on openrouter.ai and update via `supabase secrets set` |
| `Search error: function match_thoughts does not exist` | pgvector RPC not deployed | Run the setup SQL from `docs/01-getting-started.md` against your Supabase project |
| `Failed to capture: ...upsert_thought...` | `upsert_thought` RPC missing | Same as above — ensure all required database functions are deployed |
| Claude Desktop tools not appearing | Connector URL or key is wrong | Re-paste the URL in Settings → Connectors; confirm with a direct `curl` first |
| `duplex` TypeScript error in local dev | Deno type mismatch on `Request` options | Expected — the `@ts-ignore` in `index.ts` suppresses this for Deno's streaming body requirement |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this module
- [../docs/01-getting-started.md](../docs/01-getting-started.md) — End-to-end setup including database RPCs and connector configuration
- [../integrations/kubernetes-deployment/](../integrations/kubernetes-deployment/) — Kubernetes deployment of this server
