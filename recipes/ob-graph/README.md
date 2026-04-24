# ob-graph

> Knowledge graph layer for Open Brain — adds nodes, edges, and traversal tools as a Supabase Edge Function MCP server on top of the `thoughts` table.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Your Supabase project URL (`https://<ref>.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key for bypassing RLS in the Edge Function | Yes |
| `MCP_ACCESS_KEY` | Shared secret for authenticating MCP requests | Yes |
| `DEFAULT_USER_ID` | UUID of the user whose graph data this function operates on | Yes |

### HTTP Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `*` | Health check — returns `{"status":"ok","service":"OB-Graph MCP","version":"1.0.0"}` |
| `POST` | `*` | MCP protocol entry point — all tool calls arrive here |

Authentication is enforced on `POST` via a `key` query param or `x-access-key` header matched against `MCP_ACCESS_KEY`.

```bash
# Health check
curl https://<ref>.supabase.co/functions/v1/ob-graph

# Send an MCP tool call (key as query param)
curl -X POST \
  "https://<ref>.supabase.co/functions/v1/ob-graph?key=YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_nodes","arguments":{"query":"Supabase"}}}'
```

### MCP Tools

| Tool | Description |
|------|-------------|
| `create_node` | Add a node (person, project, concept, tool, place, etc.) |
| `create_edge` | Create a directed relationship between two nodes |
| `search_nodes` | Find nodes by label or type (case-insensitive) |
| `get_neighbors` | List all nodes directly connected to a given node |
| `traverse_graph` | Walk the graph N hops from a starting node |
| `find_path` | Find shortest path between two nodes (BFS) |
| `update_node` | Update label, type, or merge new properties into a node |
| `delete_node` | Remove a node and cascade-delete all its edges |
| `delete_edge` | Remove a specific edge by ID |
| `list_edge_types` | List all relationship types in use with counts |

### Commands

```bash
# Apply the schema to your Supabase project
supabase db push
# or run schema.sql directly via the Supabase Dashboard SQL Editor

# Deploy the Edge Function
supabase functions deploy ob-graph

# Set secrets (one-time, per project)
supabase secrets set SUPABASE_URL=https://<ref>.supabase.co
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=<key>
supabase secrets set MCP_ACCESS_KEY=<key>
supabase secrets set DEFAULT_USER_ID=<uuid>

# Run locally for development
supabase functions serve ob-graph --env-file .env
```

### Configuration

| File | Purpose |
|------|---------|
| `.env.example` | Template for required environment variables |
| `deno.json` | Deno import map — pins all npm dependency versions |
| `schema.sql` | Database schema: tables, indexes, RLS policies, SQL functions |

### Database Tables

| Table | Purpose |
|-------|---------|
| `graph_nodes` | Entities in the knowledge graph (label, type, JSONB properties, optional `thought_id` link) |
| `graph_edges` | Directed relationships between nodes (type, weight 0–1+, JSONB properties) |

Both tables have RLS enabled. Unique constraint on `graph_edges` prevents duplicate edges of the same type between the same node pair.

**SQL Functions (called via `supabase.rpc`):**

| Function | Description |
|----------|-------------|
| `traverse_graph(p_user_id, p_start_node_id, p_max_depth, p_relationship_type)` | Recursive CTE walk — returns reachable nodes with depth and path |
| `find_shortest_path(p_user_id, p_start_node_id, p_end_node_id, p_max_depth)` | BFS shortest path — returns ordered steps with relationship labels |

### Prerequisites

- Supabase project with the Open Brain `thoughts` table already set up
- Supabase CLI (for deploying and managing secrets)
- Deno (for local development / `supabase functions serve`)

## Common Tasks

### Deploy for the first time

```bash
# 1. Apply schema
supabase db push

# 2. Deploy function
supabase functions deploy ob-graph

# 3. Set secrets
supabase secrets set SUPABASE_URL=https://<ref>.supabase.co
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=<service-role-key>
supabase secrets set MCP_ACCESS_KEY=<any-strong-secret>
supabase secrets set DEFAULT_USER_ID=<your-supabase-auth-user-uuid>
```

### Connect to Claude Desktop

In Claude Desktop: Settings → Connectors → Add custom connector

- **URL:** `https://<ref>.supabase.co/functions/v1/ob-graph?key=YOUR_MCP_ACCESS_KEY`

### Build a simple graph from thoughts

```
# In Claude with the connector active:
"Create a node called 'Project Alpha' of type 'project'"
"Create a node called 'Alice' of type 'person'"
"Create an edge from <node-id-alice> to <node-id-project> with relationship 'works_on'"
"Traverse the graph from <node-id-alice> up to 3 hops"
```

### Find how two entities are related

```
"Find the path between <node-id-A> and <node-id-B>"
```

### Link a node to an existing thought

Pass `thought_id` when calling `create_node` — this lets you overlay graph structure on top of your existing Open Brain data without modifying the `thoughts` table.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `{"error":"Unauthorized"}` on POST | `key` param or `x-access-key` header missing or wrong | Verify `MCP_ACCESS_KEY` secret matches the value in your connector URL |
| `{"error":"DEFAULT_USER_ID not configured"}` | Secret not set on the Edge Function | Run `supabase secrets set DEFAULT_USER_ID=<uuid>` and redeploy |
| `traverse_graph` or `find_shortest_path` RPC errors | SQL functions not applied | Run `schema.sql` against your Supabase project via Dashboard or `supabase db push` |
| Duplicate edge error | Same source/target/type combination already exists | Use `list_edge_types` and `get_neighbors` to inspect before inserting |
| No path found between nodes | Nodes are not connected within `max_depth` hops | Increase `max_depth` or use `traverse_graph` to inspect reachable subgraph |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [../README.md](../README.md) — Recipes overview
