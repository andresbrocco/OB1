# vercel-neon-telegram

> Alternative Open Brain stack using Vercel + Neon (serverless Postgres) + Telegram bot for capture, with MCP endpoint and REST capture API managed via Drizzle migrations.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `DATABASE_URL` | Neon serverless Postgres connection string (with `sslmode=require`) | Yes |
| `OPENAI_API_KEY` | OpenAI API key for embeddings and classification | Yes |
| `BRAIN_ACCESS_KEY` | Bearer token for REST API and MCP auth (generate with `npm run generate-key`) | Yes |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token from @BotFather | Optional |
| `TELEGRAM_WEBHOOK_SECRET` | Secret token to validate incoming Telegram webhook requests | Optional |
| `APP_URL` | Deployed Vercel URL — used when registering the Telegram webhook | Optional |

### API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/health` | None | Liveness check — returns `{ status: "ok", timestamp }` |
| `POST` | `/api/capture` | Bearer `BRAIN_ACCESS_KEY` | Capture a thought via REST |
| `POST` | `/api/mcp` | Bearer `BRAIN_ACCESS_KEY` | MCP protocol endpoint (tools: `capture_thought`, `search_thoughts`, `list_thoughts`) |
| `GET` | `/api/mcp` | None | Returns 405 — POST only |
| `POST` | `/api/telegram` | Webhook secret header | Telegram bot webhook receiver |

```bash
# Health check
curl https://your-project.vercel.app/api/health

# Capture a thought
curl -X POST https://your-project.vercel.app/api/capture \
  -H "Authorization: Bearer $BRAIN_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": "My thought here", "source": "api"}'

# MCP — capture via tool call
curl -X POST https://your-project.vercel.app/api/mcp \
  -H "Authorization: Bearer $BRAIN_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"capture_thought","arguments":{"content":"My thought"}}}'

# MCP — semantic search
curl -X POST https://your-project.vercel.app/api/mcp \
  -H "Authorization: Bearer $BRAIN_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"search_thoughts","arguments":{"query":"project ideas","limit":5}}}'
```

### Commands

```bash
# Development
npm run dev                   # Start Next.js dev server on http://localhost:3000

# Build & production
npm run build                 # Build Next.js app for production
npm run start                 # Start production server locally

# Database
npm run migrate               # Run SQL migrations against Neon (requires DATABASE_URL)

# Setup
npm run generate-key          # Generate a secure BRAIN_ACCESS_KEY value
npm run set-telegram-webhook  # Register /api/telegram with Telegram (requires TELEGRAM_BOT_TOKEN + APP_URL)

# Testing
npm run test                  # Run Vitest unit tests
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for all required environment variables |
| `vercel.json` | Vercel deployment configuration |
| `next.config.mjs` | Next.js configuration |
| `tsconfig.json` | TypeScript compiler settings |

### Database Tables

| Table | Purpose |
|-------|---------|
| `thoughts` | Core thought storage — `id`, `content`, `embedding` (1536-dim vector), `metadata` (JSONB), `source`, `created_at`, `updated_at` |

**SQL migrations** are in `sql/`:

| File | Description |
|------|-------------|
| `sql/001-create-thoughts.sql` | Creates `thoughts` table with HNSW vector index, GIN metadata index, and `updated_at` trigger |
| `sql/002-match-thoughts.sql` | Creates `match_thoughts()` pgvector function for semantic similarity search |

### MCP Tools

| Tool | Description |
|------|-------------|
| `capture_thought` | Save a thought — auto-embeds and classifies (max 10,000 chars) |
| `search_thoughts` | Semantic vector search with optional `type`, `topic`, `threshold`, and `limit` filters |
| `list_thoughts` | Chronological browse with optional `type`, `topic`, and `since` filters |

### Telegram Bot Commands

| Command | Description |
|---------|-------------|
| `/start` | Show welcome message and command list |
| `/search <query>` | Semantic search returning top 5 results |
| (plain text) | Capture any message as a thought |
| (photo + caption) | Capture caption as a thought |

### Prerequisites

- Node.js 18+ and npm
- [Neon](https://console.neon.tech) project with the `pgvector` extension enabled
- OpenAI API key with access to `text-embedding-ada-002`
- Vercel account (for deployment)
- Telegram bot token from @BotFather (optional — only needed for Telegram capture)

## Common Tasks

### First-time Setup

```bash
# 1. Install dependencies
npm install

# 2. Copy and fill environment variables
cp .env.example .env.local

# 3. Generate a secure access key and add it to .env.local
npm run generate-key

# 4. Run database migrations against your Neon instance
npm run migrate

# 5. Start local dev server
npm run dev
```

### Deploy to Vercel

```bash
# Push to GitHub and import the repo in Vercel dashboard, or use the CLI:
npx vercel deploy

# Add all env vars in Vercel project settings, then redeploy for them to take effect.
```

### Register the Telegram Webhook (after first Vercel deploy)

```bash
# Set APP_URL to your live Vercel URL first, then:
APP_URL=https://your-project.vercel.app \
TELEGRAM_BOT_TOKEN=your-token \
TELEGRAM_WEBHOOK_SECRET=your-secret \
npm run set-telegram-webhook
```

### Connect Claude Desktop via MCP

In Claude Desktop settings, add a custom connector pointing to:

```
https://your-project.vercel.app/api/mcp
```

Use `BRAIN_ACCESS_KEY` as the bearer token in the connector auth header.

### Run Tests

```bash
npm run test
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `401 Unauthorized` on `/api/capture` or `/api/mcp` | Missing or wrong `Authorization: Bearer` header | Set `BRAIN_ACCESS_KEY` in env and pass it as `Authorization: Bearer <key>` |
| `503` on `/api/telegram` | `TELEGRAM_BOT_TOKEN` or `TELEGRAM_WEBHOOK_SECRET` not set | Add both vars to Vercel env settings and redeploy |
| `401 Invalid webhook secret` on Telegram updates | Mismatch between registered secret and `TELEGRAM_WEBHOOK_SECRET` | Re-run `npm run set-telegram-webhook` after updating the secret |
| `429 Rate limit exceeded` on capture | In-memory rate limiter triggered (per serverless instance) | Wait ~60 seconds; the `Retry-After: 60` header confirms the window |
| Migration fails with "extension not found" | `pgvector` not enabled on Neon project | Enable the `vector` extension in the Neon console before running `npm run migrate` |
| MCP endpoint returns `405` on GET | Expected — MCP only accepts POST | Use `POST` for all MCP tool calls |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [src/app/api/CONTEXT.md](src/app/api/CONTEXT.md) — API route architecture
- [src/lib/CONTEXT.md](src/lib/CONTEXT.md) — Library layer context
- [src/CONTEXT.md](src/CONTEXT.md) — Source structure context
