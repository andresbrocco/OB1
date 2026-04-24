# Home Maintenance Tracker

> Extension 2 of the Open Brain learning path: deploys a Supabase Edge Function MCP server for tracking recurring maintenance tasks, logging completed work, and surfacing upcoming items.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Your Supabase project URL (`https://<project-ref>.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key — set automatically on Edge Function deploy | Yes |
| `MCP_ACCESS_KEY` | Secret key used to authenticate MCP requests — must be set manually | Yes |
| `DEFAULT_USER_ID` | UUID of the user whose data this function operates on | Yes |

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically when you deploy via the Supabase CLI. You must set `MCP_ACCESS_KEY` and `DEFAULT_USER_ID` as Edge Function secrets.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/*` | MCP protocol handler — receives all tool calls from Claude Desktop |
| `GET` | `/*` | Health check — returns service name and version |

Authentication: pass your `MCP_ACCESS_KEY` as either the `key` query parameter or the `x-access-key` header.

```bash
# Health check
curl https://<project-ref>.supabase.co/functions/v1/home-maintenance

# MCP tool call (example — Claude Desktop sends this automatically)
curl -X POST \
  "https://<project-ref>.supabase.co/functions/v1/home-maintenance?key=YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"get_upcoming_maintenance","arguments":{"days_ahead":30}},"id":1}'
```

### MCP Tools

| Tool | Description |
|------|-------------|
| `add_maintenance_task` | Create a recurring or one-time maintenance task |
| `log_maintenance` | Record that a task was completed; auto-updates `last_completed` and `next_due` |
| `get_upcoming_maintenance` | List tasks due within the next N days (default 30) |
| `search_maintenance_history` | Search logs by task name, category, or date range |

### Commands

```bash
# Apply the schema to your Supabase database
supabase db push
# or run schema.sql directly in the Supabase SQL editor

# Deploy the Edge Function
supabase functions deploy home-maintenance

# Set required secrets (run once after deploying)
supabase secrets set MCP_ACCESS_KEY=your-secret-key
supabase secrets set DEFAULT_USER_ID=your-user-uuid

# Serve locally for testing (requires Supabase CLI)
supabase functions serve home-maintenance --env-file .env
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables |
| `deno.json` | Deno import map — pins all npm dependencies |
| `schema.sql` | Database schema — run once before deploying the function |

### Database Tables

| Table | Purpose |
|-------|---------|
| `maintenance_tasks` | Recurring and one-time maintenance items with scheduling fields (`frequency_days`, `next_due`, `last_completed`) |
| `maintenance_logs` | Immutable history of completed work, including cost, performer, and contractor notes |

Both tables have Row Level Security (RLS) enabled. A database trigger on `maintenance_logs` automatically recalculates `next_due` on the parent `maintenance_tasks` row after each log insert.

### Prerequisites

- Supabase project with the core Open Brain `thoughts` table already set up
- Supabase CLI installed and linked to your project (`supabase login`, `supabase link`)
- Deno runtime (used by Supabase Edge Functions)
- Claude Desktop with a custom connector configured

## Common Tasks

### Deploy for the first time

```bash
# 1. Apply the schema
supabase db push
# or paste schema.sql into the Supabase SQL editor and run it

# 2. Deploy the function
supabase functions deploy home-maintenance

# 3. Set secrets
supabase secrets set MCP_ACCESS_KEY=your-secret-key
supabase secrets set DEFAULT_USER_ID=your-supabase-user-uuid

# 4. Get your function URL
# Format: https://<project-ref>.supabase.co/functions/v1/home-maintenance
```

### Connect to Claude Desktop

1. Open Claude Desktop → Settings → Connectors → Add custom connector
2. Enter your function URL with your access key appended: `https://<project-ref>.supabase.co/functions/v1/home-maintenance?key=YOUR_MCP_ACCESS_KEY`
3. Save. Claude will now have access to all four maintenance tools.

### Add a recurring task via Claude

Ask Claude: "Add a quarterly HVAC filter replacement task, due in 90 days, medium priority."

Claude will call `add_maintenance_task` with `frequency_days: 90`.

### Log completed work and update schedule

Ask Claude: "Log that I just replaced the HVAC filter. Cost was $18, done by myself."

Claude will call `log_maintenance`. The database trigger fires and sets the next due date automatically based on `frequency_days`.

### Check what is coming up

Ask Claude: "What home maintenance do I have due in the next 60 days?"

Claude will call `get_upcoming_maintenance` with `days_ahead: 60`.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `401 Unauthorized` on all requests | `MCP_ACCESS_KEY` missing or mismatch | Run `supabase secrets set MCP_ACCESS_KEY=...` and redeploy |
| `500 DEFAULT_USER_ID not configured` | `DEFAULT_USER_ID` secret not set | Run `supabase secrets set DEFAULT_USER_ID=<uuid>` |
| Tools not appearing in Claude | Connector URL wrong or function not deployed | Verify the URL returns `{"status":"ok"}` via `curl`, then re-add connector |
| `next_due` not updating after logging | Trigger not created | Re-run `schema.sql` — the trigger `update_task_after_log` must exist |
| RLS blocking queries | Service role key not in use | Confirm `SUPABASE_SERVICE_ROLE_KEY` is set (not the anon key) |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture and design context
- [../README.md](../README.md) — Extensions category overview
- [../../primitives/deploy-edge-function/README.md](../../primitives/deploy-edge-function/README.md) — Edge Function deployment pattern
- [../../primitives/remote-mcp/README.md](../../primitives/remote-mcp/README.md) — Remote MCP connection pattern
