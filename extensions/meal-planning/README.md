# Meal Planning

> Extension 4 of the Open Brain learning path: adds recipe tracking, weekly meal planning, and shared household shopping lists via a Supabase Edge Function MCP server.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Supabase project URL | — | Yes (auto-set by Supabase) |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key for the primary MCP server | — | Yes (auto-set by Supabase) |
| `MCP_ACCESS_KEY` | Secret key for authenticating Claude Desktop requests to the primary server | — | Yes (set manually) |
| `DEFAULT_USER_ID` | UUID of the owner user; scopes all writes to this account | — | Yes (set manually) |
| `SUPABASE_HOUSEHOLD_KEY` | Separate service role key for the shared household server | — | Yes, for `shared-server.ts` |
| `MCP_HOUSEHOLD_ACCESS_KEY` | Secret key for authenticating household member requests | — | Yes, for `shared-server.ts` |

> `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically when you deploy via the Supabase CLI. You must set `MCP_ACCESS_KEY`, `DEFAULT_USER_ID`, `SUPABASE_HOUSEHOLD_KEY`, and `MCP_HOUSEHOLD_ACCESS_KEY` as Edge Function secrets manually.

### MCP Tools — Primary Server (`index.ts`)

The primary server exposes 6 tools over a single `POST *` route, secured by `MCP_ACCESS_KEY`.

| Tool | Description |
|------|-------------|
| `add_recipe` | Add a recipe with ingredients, instructions, tags, and rating |
| `search_recipes` | Search recipes by name, cuisine, tag, or ingredient |
| `update_recipe` | Update any field of an existing recipe by UUID |
| `create_meal_plan` | Plan meals for a full week (breakfast/lunch/dinner/snack per day) |
| `get_meal_plan` | Retrieve a week's meal plan with joined recipe details |
| `generate_shopping_list` | Auto-aggregate ingredients from a week's recipes into a shopping list |

### MCP Tools — Shared Server (`shared-server.ts`)

The shared server exposes 4 read-focused tools over `POST /mcp`, secured by `MCP_HOUSEHOLD_ACCESS_KEY`.

| Tool | Description |
|------|-------------|
| `view_meal_plan` | View a week's meal plan (read-only) |
| `view_recipes` | Browse or search recipes (read-only) |
| `view_shopping_list` | View the shopping list for a given week |
| `mark_item_purchased` | Toggle an item's purchased status on a shopping list |

### Health Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `*` | Primary server health check — returns `{ status: "ok", service: "Meal Planning", version: "1.0.0" }` |
| `GET` | `/` | Shared server health check — returns `{ status: "ok", service: "Meal Planning (Shared)", version: "1.0.0" }` |

```bash
# Verify primary server is live
curl https://<project-ref>.supabase.co/functions/v1/meal-planning

# Verify shared server is live (if deployed as a separate function)
curl https://<project-ref>.supabase.co/functions/v1/meal-planning-shared/
```

### Authentication

All MCP requests require the access key passed as either a query parameter or a header:

```bash
# Query parameter
curl -X POST "https://<project-ref>.supabase.co/functions/v1/meal-planning?key=<MCP_ACCESS_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'

# Header
curl -X POST "https://<project-ref>.supabase.co/functions/v1/meal-planning" \
  -H "x-access-key: <MCP_ACCESS_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

### Commands

```bash
# Deploy the primary MCP server
supabase functions deploy meal-planning

# Deploy the shared household server (deploy as a separate function name)
supabase functions deploy meal-planning-shared --file extensions/meal-planning/shared-server.ts

# Set required secrets
supabase secrets set MCP_ACCESS_KEY=your-secret-key
supabase secrets set DEFAULT_USER_ID=your-supabase-user-uuid
supabase secrets set MCP_HOUSEHOLD_ACCESS_KEY=your-household-secret-key
supabase secrets set SUPABASE_HOUSEHOLD_KEY=your-household-service-role-key

# Apply the schema to your Supabase project
supabase db push
# or run schema.sql directly in the Supabase SQL editor
```

### Configuration

| File | Purpose |
|------|---------|
| `deno.json` | Deno import map — pins npm package versions for Hono, MCP SDK, Supabase JS, Zod |
| `.env.example` | Documents the three required env vars for local reference |
| `schema.sql` | Creates `recipes`, `meal_plans`, and `shopping_lists` tables with RLS policies |

### Database Tables

| Table | Purpose |
|-------|---------|
| `recipes` | Stores recipe records: name, cuisine, times, JSONB ingredients + instructions, tags, rating |
| `meal_plans` | One row per meal slot per day per week; references `recipes` via `recipe_id` |
| `shopping_lists` | One record per week; stores aggregated items as JSONB with `purchased` flag per item |

All three tables have Row Level Security enabled. The primary owner can CRUD their own rows. Household members (JWT role `household_member`) can SELECT from all tables and UPDATE `shopping_lists`.

### Prerequisites

- Supabase project with the core Open Brain `thoughts` table already set up
- Supabase CLI installed and linked to your project (`supabase link`)
- Primitives read/understood: `deploy-edge-function`, `remote-mcp`, `rls`, `shared-mcp`
- Claude Desktop configured to connect via Settings → Connectors → Add custom connector

## Common Tasks

### Add a Recipe

Ask Claude (with the connector active):

> "Add a recipe for spaghetti bolognese — Italian, 15 min prep, 45 min cook, serves 4. Ingredients: 400g pasta, 500g beef mince, 1 can tomatoes. Instructions: boil pasta, brown meat, simmer sauce, combine."

Or call the tool directly in an MCP test client:

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "add_recipe",
    "arguments": {
      "name": "Spaghetti Bolognese",
      "cuisine": "Italian",
      "prep_time_minutes": 15,
      "cook_time_minutes": 45,
      "servings": 4,
      "ingredients": [
        { "name": "pasta", "quantity": "400", "unit": "g" },
        { "name": "beef mince", "quantity": "500", "unit": "g" },
        { "name": "canned tomatoes", "quantity": "1", "unit": "can" }
      ],
      "instructions": ["Boil pasta", "Brown the meat", "Simmer sauce 30 min", "Combine and serve"]
    }
  },
  "id": 1
}
```

### Plan a Week of Meals

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "create_meal_plan",
    "arguments": {
      "week_start": "2026-04-27",
      "meals": [
        { "day_of_week": "monday", "meal_type": "dinner", "recipe_id": "<uuid>", "servings": 4 },
        { "day_of_week": "tuesday", "meal_type": "dinner", "custom_meal": "Takeout", "notes": "Pizza night" }
      ]
    }
  },
  "id": 2
}
```

### Generate a Shopping List from the Week's Plan

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "generate_shopping_list",
    "arguments": { "week_start": "2026-04-27" }
  },
  "id": 3
}
```

This aggregates ingredients from all recipe-linked meals into a single `shopping_lists` row. Calling it again on the same week updates the existing record.

### Connect the Shared Server for a Household Member

1. Deploy `shared-server.ts` as its own Edge Function (e.g., `meal-planning-shared`).
2. Set `MCP_HOUSEHOLD_ACCESS_KEY` and `SUPABASE_HOUSEHOLD_KEY` as secrets on that function.
3. Give the household member the function URL and the `MCP_HOUSEHOLD_ACCESS_KEY` value.
4. They add it as a custom connector in their Claude Desktop.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `401 Unauthorized` on every request | `MCP_ACCESS_KEY` secret not set, or key value mismatch | Run `supabase secrets set MCP_ACCESS_KEY=...` and redeploy |
| `500 DEFAULT_USER_ID not configured` | `DEFAULT_USER_ID` secret missing | Run `supabase secrets set DEFAULT_USER_ID=<your-uuid>` and redeploy |
| `relation "recipes" does not exist` | `schema.sql` has not been applied | Run `schema.sql` in the Supabase SQL editor or via `supabase db push` |
| Shopping list not updating ingredients | `generate_shopping_list` finds no recipe-linked meals | Ensure `create_meal_plan` entries use `recipe_id`, not only `custom_meal` |
| Household member gets empty results | RLS policy not matching JWT claims | Verify the household Supabase key grants the `household_member` JWT role claim |
| Claude Desktop connector silently fails | Missing `Accept: text/event-stream` header | Already handled in `index.ts` via the header-patching workaround; confirm you are using the deployed version |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this extension
- [../../primitives/deploy-edge-function/README.md](../../primitives/deploy-edge-function/README.md) — How to deploy Edge Functions
- [../../primitives/remote-mcp/README.md](../../primitives/remote-mcp/README.md) — Remote MCP server pattern
- [../../primitives/rls/README.md](../../primitives/rls/README.md) — Row Level Security pattern
- [../../primitives/shared-mcp/README.md](../../primitives/shared-mcp/README.md) — Shared household MCP pattern
- [../README.md](../README.md) — Extensions learning path overview
