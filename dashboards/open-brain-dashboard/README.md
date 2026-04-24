# open-brain-dashboard

> SvelteKit dashboard for Open Brain — read-only semantic search and thought browsing UI with a server-side MCP proxy to keep credentials out of the browser.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `MCP_URL` | URL of the Open Brain MCP Edge Function (server-only, not exposed to browser) | Yes* |
| `MCP_KEY` | Access key for the MCP Edge Function (server-only, not exposed to browser) | Yes* |
| `PUBLIC_SUPABASE_URL` | Supabase project URL — used for client-side auth | Yes |
| `PUBLIC_SUPABASE_ANON_KEY` | Supabase anon key — used for client-side auth | Yes |

> `PUBLIC_MCP_URL` / `PUBLIC_MCP_KEY` are accepted as a backward-compatible fallback but will be visible in the browser bundle. Prefer the server-only `MCP_URL` / `MCP_KEY` pair.

Copy `.env.example` to `.env.local` before running locally:

```bash
cp .env.example .env.local
# then fill in values in .env.local
```

SvelteKit only picks up env changes after restarting `npm run dev`.

**Symlink tip** — share one env file across all dashboards in the repo:

```bash
ln -s ../../.env.local dashboards/open-brain-dashboard/.env.local
```

### API Endpoints

| Method | Path | Auth required | Description |
|--------|------|---------------|-------------|
| `POST` | `/api/mcp` | Yes (Supabase session) | Server-side proxy — forwards MCP tool calls to the upstream Edge Function and returns the result |
| `GET` | `/signout` | — | Signs the current user out via Supabase Auth and redirects to `/signin` |

**MCP proxy — example request**

```bash
# Requires a valid Supabase session cookie (obtained after signing in)
curl -X POST http://localhost:5173/api/mcp \
  -H "Content-Type: application/json" \
  -b "<your-session-cookie>" \
  -d '{"name": "search_thoughts", "args": {"query": "machine learning"}}'
```

Response shape:

```json
{ "result": { ... } }
```

Error shape:

```json
{ "error": "reason string" }
```

### Commands

```bash
# Install dependencies
npm install

# Start development server (default: http://localhost:5173)
npm run dev

# Type-check all Svelte and TypeScript files
npm run check

# Watch mode type-checking
npm run check:watch

# Production build
npm run build

# Preview production build locally
npm run preview
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables — copy to `.env.local` |
| `svelte.config.js` | SvelteKit config — uses `@sveltejs/adapter-vercel` targeting Node 22 |
| `vite.config.ts` | Vite build config |
| `tsconfig.json` | TypeScript compiler options |

### Prerequisites

- Node.js 22.x (matches Vercel runtime configured in `svelte.config.js`)
- A running Open Brain Supabase project with the MCP Edge Function deployed
- Supabase project URL and anon key
- MCP URL and access key

## Common Tasks

### Run the Dashboard Locally

```bash
cd dashboards/open-brain-dashboard
cp .env.example .env.local
# Edit .env.local with your real values
npm install
npm run dev
# Open http://localhost:5173
```

### Deploy to Vercel

```bash
# From the dashboard directory
npm run build
# Then push to a Vercel-connected branch, or use:
vercel --cwd dashboards/open-brain-dashboard
```

The adapter is pre-configured for Vercel (`@sveltejs/adapter-vercel`, Node 22). Set the four environment variables in your Vercel project settings before deploying.

### Call an MCP Tool Programmatically

The `/api/mcp` route accepts a JSON body with `name` (tool name) and optional `args` (arguments object). The server resolves credentials, calls the upstream MCP Edge Function via JSON-RPC 2.0, and returns the unwrapped result.

```bash
curl -X POST https://<your-vercel-domain>/api/mcp \
  -H "Content-Type: application/json" \
  -b "<session-cookie>" \
  -d '{"name": "recall_memories", "args": {"query": "project alpha"}}'
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `Missing MCP_URL/MCP_KEY` error on `/api/mcp` | Env vars not set or not picked up | Confirm `.env.local` exists with values and restart `npm run dev` |
| Redirected to `/signin` on every page load | Supabase session not established | Check `PUBLIC_SUPABASE_URL` and `PUBLIC_SUPABASE_ANON_KEY` are correct |
| `MCP upstream HTTP 401` from the proxy | Wrong or missing `MCP_KEY` | Verify the key matches what was set when deploying the MCP Edge Function |
| `MCP upstream HTTP 502` from the proxy | MCP Edge Function is unreachable or returned an error | Check `MCP_URL` points to the correct deployed function URL |
| Env variable changes not reflected | SvelteKit caches env at startup | Restart `npm run dev` after editing `.env.local` |
| Type errors on `svelte-check` | Svelte/TS types out of sync | Run `npm run prepare` (runs `svelte-kit sync`) then retry `npm run check` |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this dashboard
- [src/CONTEXT.md](src/CONTEXT.md) — Source layout context
- [src/routes/CONTEXT.md](src/routes/CONTEXT.md) — Route architecture context
- [src/lib/CONTEXT.md](src/lib/CONTEXT.md) — Library utilities context
- [../open-brain-dashboard-next/README.md](../open-brain-dashboard-next/README.md) — Next-generation dashboard
