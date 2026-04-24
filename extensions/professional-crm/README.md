# Professional CRM

> Extension for managing professional contacts, interactions, and opportunities as a remote MCP Edge Function on Supabase.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Your Supabase project URL (set automatically on deploy) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (set automatically on deploy) | Yes |
| `MCP_ACCESS_KEY` | Secret key used to authenticate MCP requests | Yes |
| `DEFAULT_USER_ID` | UUID of the user whose data this Edge Function serves | Yes |

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically by Supabase when you deploy an Edge Function. You must set `MCP_ACCESS_KEY` and `DEFAULT_USER_ID` manually as Edge Function secrets.

### API Endpoints

This module exposes a single remote MCP server over HTTP. All MCP tool calls use `POST`; a `GET` health check is also available.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/*` | MCP endpoint — accepts all MCP JSON-RPC tool calls |
| `GET` | `/*` | Health check — returns service status |

Authentication is required on every `POST` request. Pass the `MCP_ACCESS_KEY` value as either:
- Query parameter: `?key=<MCP_ACCESS_KEY>`
- Header: `x-access-key: <MCP_ACCESS_KEY>`

```bash
# Health check
curl https://<project>.supabase.co/functions/v1/professional-crm

# Example MCP tool call (add a contact)
curl -X POST \
  "https://<project>.supabase.co/functions/v1/professional-crm?key=<MCP_ACCESS_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "add_professional_contact",
      "arguments": {
        "name": "Sarah Chen",
        "company": "DataCorp",
        "title": "VP of Engineering",
        "email": "sarah@datacorp.com",
        "how_we_met": "AI Summit 2026",
        "tags": ["ai", "engineering"]
      }
    }
  }'
```

### MCP Tools

| Tool | Description |
|------|-------------|
| `add_professional_contact` | Add a new contact with name, company, title, email, phone, LinkedIn URL, tags, and notes |
| `search_contacts` | Search contacts by free-text query or filter by tags |
| `log_interaction` | Log a touchpoint (meeting, email, call, coffee, event, LinkedIn, other); auto-updates `last_contacted` |
| `get_contact_history` | Retrieve a contact's full profile, all interactions, and linked opportunities |
| `create_opportunity` | Create a deal or collaboration opportunity, optionally linked to a contact |
| `get_follow_ups_due` | List contacts with overdue or upcoming follow-up dates (default: next 7 days) |
| `update_professional_contact` | Update any field on an existing contact, including setting or clearing `follow_up_date` |
| `link_thought_to_contact` | Cross-extension bridge: append a core Open Brain thought to a contact's notes |

### Commands

```bash
# Deploy to Supabase Edge Functions
supabase functions deploy professional-crm

# Set required secrets (run once, or when rotating keys)
supabase secrets set MCP_ACCESS_KEY=<your-key>
supabase secrets set DEFAULT_USER_ID=<your-user-uuid>

# Stream function logs
supabase functions logs professional-crm --tail
```

### Configuration

| File | Purpose |
|------|---------|
| `deno.json` | Deno import map — pins all npm dependency versions |
| `.env.example` | Documents required environment variable names |

### Database Tables

Apply `schema.sql` to your Supabase database before deploying.

| Table | Purpose |
|-------|---------|
| `professional_contacts` | Core contact records with tags, follow-up date, and `last_contacted` timestamp |
| `contact_interactions` | Interaction log (meetings, calls, emails, etc.) linked to contacts |
| `opportunities` | Deal/pipeline records, optionally linked to a contact |

Triggers in `schema.sql` automatically:
- Update `updated_at` on `professional_contacts` and `opportunities` when a row is modified.
- Update `last_contacted` on a contact whenever a new `contact_interactions` row is inserted.

Row Level Security (RLS) is enabled on all three tables; each user can only access their own rows.

### Prerequisites

- Supabase project with the Supabase CLI installed (`supabase` >= 1.x)
- `schema.sql` applied to the project database
- Deno runtime (handled automatically by Supabase Edge Functions infrastructure)
- `DEFAULT_USER_ID` set to a valid UUID that exists in your auth system

## Common Tasks

### Add a New Contact

```bash
curl -X POST \
  "https://<project>.supabase.co/functions/v1/professional-crm?key=<MCP_ACCESS_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 1,
    "method": "tools/call",
    "params": {
      "name": "add_professional_contact",
      "arguments": { "name": "Alex Rivera", "company": "Acme", "tags": ["sales"] }
    }
  }'
```

### Log an Interaction

```bash
curl -X POST \
  "https://<project>.supabase.co/functions/v1/professional-crm?key=<MCP_ACCESS_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 2,
    "method": "tools/call",
    "params": {
      "name": "log_interaction",
      "arguments": {
        "contact_id": "<uuid>",
        "interaction_type": "coffee",
        "summary": "Discussed Q3 roadmap. Intro to product team next week.",
        "follow_up_needed": true,
        "follow_up_notes": "Send intro email by Friday"
      }
    }
  }'
```

### Check Follow-Ups Due This Week

```bash
curl -X POST \
  "https://<project>.supabase.co/functions/v1/professional-crm?key=<MCP_ACCESS_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": 3,
    "method": "tools/call",
    "params": { "name": "get_follow_ups_due", "arguments": { "days_ahead": 7 } }
  }'
```

### Connect to Claude Desktop

1. Deploy the function: `supabase functions deploy professional-crm`
2. Open Claude Desktop → Settings → Connectors → Add custom connector
3. Paste `https://<project>.supabase.co/functions/v1/professional-crm?key=<MCP_ACCESS_KEY>`

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `401 Unauthorized` on every request | `MCP_ACCESS_KEY` mismatch or missing | Verify the key in the query/header matches the secret set via `supabase secrets set MCP_ACCESS_KEY` |
| `500 DEFAULT_USER_ID not configured` | Secret not set in Edge Function environment | Run `supabase secrets set DEFAULT_USER_ID=<uuid>` and redeploy |
| `Failed to add professional contact: ...` from Supabase | `schema.sql` not yet applied, or RLS blocking insert | Apply `schema.sql` to your project database; confirm RLS policies allow the service role |
| Tools appear in Claude but return no data | `DEFAULT_USER_ID` does not match any rows | Ensure the UUID matches the user whose data was inserted |
| SSE/streaming errors from MCP client | Some connectors omit `text/event-stream` in `Accept` headers | The function patches the `Accept` header automatically; confirm you are on the latest deployed version |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this extension
- [../README.md](../README.md) — Extensions directory overview
- [../../primitives/remote-mcp/README.md](../../primitives/remote-mcp/README.md) — Remote MCP pattern reference
- [../../docs/01-getting-started.md](../../docs/01-getting-started.md) — Open Brain setup and connector wiring
