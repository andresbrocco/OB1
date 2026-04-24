# Family Calendar

> Extension 3 of the Open Brain learning path: multi-person family scheduling via an MCP Edge Function, covering activities, important dates, and household roster management.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Your Supabase project URL | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (set automatically on Edge Function deploy) | Yes |
| `MCP_ACCESS_KEY` | Shared secret sent by Claude Desktop to authenticate MCP requests | Yes |
| `DEFAULT_USER_ID` | UUID of the Supabase user whose data this function operates on | Yes |

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically when you deploy a Supabase Edge Function. You must set `MCP_ACCESS_KEY` and `DEFAULT_USER_ID` manually via the Supabase dashboard or CLI.

### MCP Endpoint

The function exposes a single HTTP route that handles all MCP traffic.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/*` | MCP JSON-RPC endpoint (all tool calls) |
| `GET` | `/*` | Health check — returns `{"status":"ok","service":"Family Calendar","version":"1.0.0"}` |

Authentication: pass `MCP_ACCESS_KEY` either as the `key` query parameter or the `x-access-key` header.

```bash
# Health check
curl https://<project-ref>.supabase.co/functions/v1/family-calendar

# Example: invoke add_family_member via MCP (key as query param)
curl -X POST \
  "https://<project-ref>.supabase.co/functions/v1/family-calendar?key=<MCP_ACCESS_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "add_family_member",
      "arguments": {"name": "Alice", "relationship": "child"}
    }
  }'
```

### MCP Tools

| Tool | Description |
|------|-------------|
| `add_family_member` | Add a person to the household roster |
| `add_activity` | Schedule a one-time or recurring activity |
| `get_week_schedule` | Retrieve all activities for a given week, grouped by day |
| `search_activities` | Search activities by title, type, or family member |
| `add_important_date` | Track a birthday, anniversary, or deadline |
| `get_upcoming_dates` | List important dates in the next N days (default 30) |

### Commands

```bash
# Deploy the Edge Function
supabase functions deploy family-calendar

# Set required secrets (run once per project)
supabase secrets set MCP_ACCESS_KEY=<your-secret>
supabase secrets set DEFAULT_USER_ID=<your-user-uuid>

# Serve locally for development
supabase functions serve family-calendar --env-file .env.example

# Apply the schema to your database
supabase db push
# or run schema.sql directly in the Supabase SQL editor
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables |
| `deno.json` | Deno import map — pins all npm dependencies |
| `schema.sql` | Database table definitions and indexes |

### Database Tables

| Table | Purpose |
|-------|---------|
| `family_members` | Household roster — names, relationships, birth dates |
| `activities` | One-time and recurring scheduled events |
| `important_dates` | Birthdays, anniversaries, and deadlines with optional yearly recurrence |

All tables are scoped by `user_id`. `activities` and `important_dates` optionally reference a `family_members` row; a `null` foreign key means the event applies to the whole family.

### Prerequisites

- Open Brain core setup complete (Supabase project with `thoughts` table)
- Supabase CLI installed and linked to your project
- [`deploy-edge-function`](../../primitives/deploy-edge-function/) primitive reviewed
- [`remote-mcp`](../../primitives/remote-mcp/) primitive reviewed
- Claude Desktop with a custom connector configured

## Common Tasks

### Add the connector to Claude Desktop

After deploying, add the Edge Function URL as a custom connector in Claude Desktop:

1. Open Claude Desktop → Settings → Connectors → Add custom connector
2. Enter: `https://<project-ref>.supabase.co/functions/v1/family-calendar?key=<MCP_ACCESS_KEY>`
3. Save. The six MCP tools will appear in your tool list.

### Register your household

Ask Claude to call `add_family_member` for each person in your household before scheduling activities. Example prompt:

```
Add my family: myself (self), partner Alex (spouse), and daughter Maya (child, born 2018-04-10).
```

### Schedule a recurring activity

```
Add Maya's soccer practice: every Tuesday and Thursday, 4:00–5:30 PM, at Riverside Fields,
starting 2026-09-02.
```

### Get this week's schedule

```
What does our family schedule look like for the week of 2026-04-27?
```

### Track upcoming birthdays

```
What important dates do we have in the next 60 days?
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `{"error":"Unauthorized"}` on every request | `MCP_ACCESS_KEY` mismatch or missing `key` param / `x-access-key` header | Confirm the secret value with `supabase secrets list` and re-check the connector URL |
| `DEFAULT_USER_ID not configured` (500) | `DEFAULT_USER_ID` secret not set | Run `supabase secrets set DEFAULT_USER_ID=<uuid>` and redeploy |
| Tool calls return empty arrays | Schema not applied | Run `schema.sql` against your Supabase project in the SQL editor |
| Claude Desktop shows no tools | Connector URL not saved or function not deployed | Re-deploy with `supabase functions deploy family-calendar` and verify the health check endpoint returns `{"status":"ok"}` |
| `activities` query returns stale data | `end_date` filter logic | Recurring events with `end_date = null` are treated as ongoing; pass an explicit `end_date` to stop them |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this extension
- [../../primitives/deploy-edge-function/](../../primitives/deploy-edge-function/) — How to deploy Supabase Edge Functions
- [../../primitives/remote-mcp/](../../primitives/remote-mcp/) — Remote MCP pattern used here
- [../household-knowledge/](../household-knowledge/) — Extension 2: household knowledge base (prerequisite)
- [../home-maintenance/](../home-maintenance/) — Extension 4: home maintenance tracking
