# Household Knowledge Base

> Extension 1 of the Open Brain learning path: deploys a Supabase Edge Function MCP server for storing and retrieving household facts — paint colors, appliance details, vendor contacts, and measurements.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Supabase project URL (`https://your-project.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (set automatically on deploy) | Yes |
| `MCP_ACCESS_KEY` | Shared secret used to authenticate MCP requests | Yes |
| `DEFAULT_USER_ID` | UUID of the user whose data this function serves | Yes |

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically when you deploy via Supabase CLI. You must set `MCP_ACCESS_KEY` and `DEFAULT_USER_ID` as Edge Function secrets manually.

### MCP Tools (via HTTP POST)

The Edge Function exposes one HTTP route that multiplexes all MCP tools.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/{function-name}` | MCP JSON-RPC — all tool calls go here |
| GET | `/{function-name}` | Health check — returns `{"status":"ok"}` |

Authentication: pass `?key=<MCP_ACCESS_KEY>` as a query parameter **or** set the `x-access-key` header.

```bash
# Health check
curl https://your-project.supabase.co/functions/v1/household-knowledge

# Call an MCP tool directly (example: list all vendors)
curl -X POST \
  "https://your-project.supabase.co/functions/v1/household-knowledge?key=YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "list_vendors",
      "arguments": {}
    }
  }'

# Search for a paint color
curl -X POST \
  "https://your-project.supabase.co/functions/v1/household-knowledge?key=YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "search_household_items",
      "arguments": {"query": "paint", "location": "Living Room"}
    }
  }'
```

### Available MCP Tools

| Tool | Description |
|------|-------------|
| `add_household_item` | Add a paint color, appliance, measurement, or document |
| `search_household_items` | Search items by name, category, or location |
| `get_item_details` | Fetch full details of one item by UUID |
| `add_vendor` | Add a service provider with contact info and rating |
| `list_vendors` | List all vendors, optionally filtered by service type |

### Commands

```bash
# Deploy the Edge Function
supabase functions deploy household-knowledge

# Set required secrets
supabase secrets set MCP_ACCESS_KEY=your-secret-key
supabase secrets set DEFAULT_USER_ID=your-user-uuid

# Run locally for testing
supabase functions serve household-knowledge --env-file .env.example

# Apply the schema to your Supabase project
supabase db push
# — or run schema.sql directly in the Supabase SQL editor
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Environment variable template |
| `deno.json` | Deno import map (Hono, MCP SDK, Supabase JS, Zod) |
| `schema.sql` | Database table definitions and RLS policies |
| `metadata.json` | Extension metadata for the OB1 registry |

### Database Tables

| Table | Purpose |
|-------|---------|
| `household_items` | Household facts — paint codes, appliance specs, measurements, documents |
| `household_vendors` | Service provider contacts with rating and last-used date |

Both tables have Row Level Security enabled. The service role key used by the Edge Function bypasses RLS; per-user scoping is enforced via `DEFAULT_USER_ID` at the application layer.

### Prerequisites

- Supabase project with Open Brain core setup complete (see `docs/01-getting-started.md`)
- Supabase CLI installed
- `schema.sql` applied to your database before first use

## Common Tasks

### Deploy for the First Time

```bash
# 1. Apply the schema
# Run schema.sql in the Supabase SQL editor, or:
supabase db push

# 2. Deploy the function
supabase functions deploy household-knowledge

# 3. Set secrets
supabase secrets set MCP_ACCESS_KEY=your-secret-key
supabase secrets set DEFAULT_USER_ID=$(supabase auth admin list-users | grep your@email.com | awk '{print $1}')
```

### Connect to Claude Desktop

1. Open Claude Desktop → Settings → Connectors → Add custom connector.
2. Paste your Edge Function URL:
   `https://your-project.supabase.co/functions/v1/household-knowledge?key=YOUR_MCP_ACCESS_KEY`
3. Save. The five household tools will appear in the tool list.

### Add a Paint Color via Claude

Once connected, prompt Claude:

> "Add a household item: Living Room paint, Sherwin Williams Sea Salt SW 6204, 2 gallons purchased March 2025."

Claude will call `add_household_item` with the appropriate fields.

### Find a Vendor

> "Who is my plumber?"

Claude will call `list_vendors` with `service_type: "plumber"`.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `401 Unauthorized` on every request | `MCP_ACCESS_KEY` mismatch or missing | Verify the key in your connector URL matches `supabase secrets set MCP_ACCESS_KEY=...` |
| `500 DEFAULT_USER_ID not configured` | Secret not set on the deployed function | Run `supabase secrets set DEFAULT_USER_ID=your-uuid` and redeploy |
| Tool calls return `relation "household_items" does not exist` | Schema not applied | Run `schema.sql` in the Supabase SQL editor |
| Claude Desktop shows no tools | Connector URL has wrong function name or region | Confirm the URL resolves with a GET (health check should return `{"status":"ok"}`) |
| `accept` header error in logs | Older Claude Desktop build | The function patches the `Accept` header automatically; update Claude Desktop if errors persist |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture and design context for this extension
- [../../primitives/deploy-edge-function/README.md](../../primitives/deploy-edge-function/README.md) — How to deploy Supabase Edge Functions
- [../../primitives/remote-mcp/README.md](../../primitives/remote-mcp/README.md) — MCP over HTTP pattern used here
- [../../docs/01-getting-started.md](../../docs/01-getting-started.md) — Open Brain core setup
