# open-brain-dashboard-next

> Next.js 14 admin dashboard for Open Brain — audit, dedup, kanban workflow, ingest pipeline UI, and semantic search over the thoughts table, protected by iron-session passphrase auth.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `NEXT_PUBLIC_API_URL` | URL of your Open Brain REST API (Supabase Edge Function) | Yes |
| `SESSION_SECRET` | 32+ character secret for iron-session cookie encryption | Yes |
| `RESTRICTED_PASSPHRASE_HASH` | SHA-256 hash of passphrase to unlock restricted/sensitive content (requires sensitivity-tiers primitive) | No |

Generate values:

```bash
# SESSION_SECRET
openssl rand -hex 32

# RESTRICTED_PASSPHRASE_HASH
echo -n "your-passphrase" | shasum -a 256
```

Copy `.env.example` to `.env.local` and fill in the required values before starting.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/audit` | Fetch thoughts for audit review |
| POST | `/api/audit` | Submit audit action on a thought |
| DELETE | `/api/audit/delete` | Delete a thought via audit |
| GET | `/api/duplicates` | List detected duplicate thought pairs |
| POST | `/api/duplicates/resolve` | Resolve a duplicate pair (keep/merge/dismiss) |
| GET | `/api/ingest` | List ingest pipeline jobs |
| GET | `/api/ingest/[id]` | Get a single ingest job by ID |
| POST | `/api/ingest/[id]/execute` | Execute an ingest job |
| GET | `/api/kanban` | Fetch kanban board state |
| POST | `/api/kanban` | Create a kanban card |
| DELETE | `/api/kanban` | Delete a kanban card |
| DELETE | `/api/kanban/delete` | Bulk delete kanban cards |
| POST | `/api/kanban/update` | Update a kanban card (move column, edit) |
| POST | `/api/logout` | Destroy the iron-session cookie |
| POST | `/api/restricted` | Verify passphrase and unlock restricted tier |
| POST | `/api/search` | Semantic search over thoughts |
| GET | `/api/thoughts/[id]/connections` | Fetch semantic connections for a thought |
| POST | `/api/thoughts/[id]/reflection` | Generate a reflection for a thought |

All API routes that modify data require an active iron-session (`open_brain_session` cookie). Unauthenticated requests receive `401`.

```bash
# Example: semantic search (session cookie required)
curl -X POST http://localhost:3000/api/search \
  -H "Content-Type: application/json" \
  -b "open_brain_session=<your-session-cookie>" \
  -d '{"query": "what did I learn about TypeScript last week"}'

# Example: fetch kanban board
curl -X GET http://localhost:3000/api/kanban \
  -b "open_brain_session=<your-session-cookie>"

# Example: resolve a duplicate pair
curl -X POST http://localhost:3000/api/duplicates/resolve \
  -H "Content-Type: application/json" \
  -b "open_brain_session=<your-session-cookie>" \
  -d '{"keep_id": "uuid-a", "delete_id": "uuid-b"}'
```

### Commands

```bash
# Development — starts Next.js dev server on http://localhost:3000
npm run dev

# Build — production build
npm run build

# Start — run the production build
npm run start

# Lint — ESLint with Next.js rules
npm run lint
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables |
| `.env.local` | Local environment overrides (gitignored) |
| `next.config.ts` | Next.js configuration |
| `middleware.ts` | Route-level session guard — redirects unauthenticated users to `/login` |
| `tsconfig.json` | TypeScript configuration |
| `eslint.config.mjs` | ESLint configuration |

### Prerequisites

- Node.js 20+
- A running Open Brain REST API (Supabase Edge Function) — set as `NEXT_PUBLIC_API_URL`
- `SESSION_SECRET` of at least 32 characters
- (Optional) sensitivity-tiers primitive deployed if using restricted content unlock

## Common Tasks

### Run Locally

```bash
cd dashboards/open-brain-dashboard-next
cp .env.example .env.local
# Edit .env.local with your values
npm install
npm run dev
# Open http://localhost:3000
```

### Log In

Navigate to `http://localhost:3000/login`. Enter your Open Brain API key (the Supabase service role key or anon key depending on your setup). The key is stored in an encrypted iron-session cookie valid for 24 hours.

### Unlock Restricted Content

If `RESTRICTED_PASSPHRASE_HASH` is set, a lock toggle appears in the UI. Submit the passphrase to unlock `sensitivity_tier = restricted` thoughts for the duration of the session.

### Deploy to Vercel

```bash
# Push to your fork, then connect to Vercel
# Set env vars in Vercel project settings:
#   NEXT_PUBLIC_API_URL
#   SESSION_SECRET
#   RESTRICTED_PASSPHRASE_HASH  (optional)
vercel --prod
```

### Generate a New SESSION_SECRET

```bash
openssl rand -hex 32
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| Server crashes on startup with `SESSION_SECRET env var is required` | `SESSION_SECRET` is missing or fewer than 32 characters | Set a 32+ character value in `.env.local` |
| All pages redirect to `/login` | Session cookie is missing or expired (24h TTL) | Log in again at `/login` |
| API returns `401 Unauthorized` | Request made without a valid session cookie | Ensure the `open_brain_session` cookie is present and not expired |
| `NEXT_PUBLIC_API_URL` not reaching Supabase | Env var not set or Edge Function not deployed | Verify the URL in `.env.local` and that the `open-brain-rest` Edge Function is live |
| Restricted thoughts not visible | `RESTRICTED_PASSPHRASE_HASH` not set, or passphrase not entered | Set the hash env var and unlock via the UI toggle |
| Duplicate detection returns no results | Embeddings not generated for thoughts | Run the embedding backfill recipe before using the duplicates view |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this dashboard
- [app/api/CONTEXT.md](app/api/CONTEXT.md) — API route architecture details
- [lib/CONTEXT.md](lib/CONTEXT.md) — Auth and utility library context
- [components/CONTEXT.md](components/CONTEXT.md) — Component architecture context
- [../open-brain-dashboard/README.md](../open-brain-dashboard/README.md) — SvelteKit predecessor dashboard
