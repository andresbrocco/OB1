# CONTEXT.md — ob-graph

## Purpose

A knowledge graph layer that adds nodes-and-edges relationship modeling to Open Brain. Exposes graph construction and traversal as an MCP server (Deno/Hono deployed as a Supabase Edge Function), backed by two PostgreSQL tables and recursive CTE SQL functions.

## Responsibility Boundaries

- **Owns**: `graph_nodes` and `graph_edges` table definitions, RLS policies for those tables, the `traverse_graph` and `find_shortest_path` PostgreSQL functions, and all 10 MCP tools that operate on them.
- **Delegates to**: Supabase for persistence and RLS enforcement; the Open Brain `thoughts` table for optional semantic linkage via `thought_id`.
- **Does not handle**: Vector/semantic search, thought ingestion, or modification of any existing Open Brain tables.

## Key Concepts

- **Node**: An entity in the graph (`graph_nodes`). Has a `label`, a `node_type` (e.g. `person`, `project`, `concept`), a flexible `properties` JSONB bag, and an optional `thought_id` foreign key that links it to an existing thought in the core Open Brain table without altering that table.
- **Edge**: A directed, typed relationship between two nodes (`graph_edges`). Has a `relationship_type` string, an optional `weight` float, and is unique per `(user_id, source, target, type)` — duplicate edges of the same type are rejected at the database level.
- **traverse_graph**: Outgoing-only depth-first walk using a recursive CTE. Cycle-safe via path-array membership check. Returns all reachable nodes up to `max_depth` hops.
- **find_shortest_path**: Bidirectional BFS using a recursive CTE that follows edges in both directions. Returns the ordered step sequence from source to target.

## Non-Obvious Details

- **`thought_id` linkage is optional and additive.** Nodes exist independently; the `thought_id` column simply lets graph nodes overlay the existing `thoughts` table without requiring a join or modifying that table's schema.
- **Claude Desktop `Accept` header workaround.** The MCP server patches incoming requests to add `Accept: application/json, text/event-stream` when missing, because Claude Desktop's connector does not send the header that `StreamableHTTPTransport` requires. This is handled at the top of the Hono request handler before any auth logic.
- **Auth uses `DEFAULT_USER_ID` env var, not `auth.uid()` at the application layer.** RLS policies do use `auth.uid()` for direct database access, but the MCP server itself authenticates via a static `MCP_ACCESS_KEY` query param or header and injects the configured `DEFAULT_USER_ID` into all queries.
- **Edge cascade deletes.** `graph_edges` has `ON DELETE CASCADE` on both `source_node_id` and `target_node_id`, so deleting a node automatically removes all its edges. The `delete_node` MCP tool relies on this rather than explicitly deleting edges.
- **`update_node` merges properties.** Rather than replacing the existing `properties` JSONB object, the tool fetches the current value and shallow-merges the incoming keys on top of it.
- **`traverse_graph` is outgoing-only; `find_shortest_path` is bidirectional.** These are different traversal strategies served by different SQL functions. `get_neighbors` at depth 1 also supports explicit `incoming`/`outgoing`/`both` direction filtering via application-level query construction rather than a SQL function.

## Related Modules

- **[integrations](../../integrations/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Symmetric relation canonical ordering, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, support_count, symmetric relation canonicalization, traverse_graph (outgoing recursive CTE))
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (derived_from edges, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[schemas](../../schemas/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, metadata-overlap thought connections, relationship_type, traverse_graph (outgoing recursive CTE))
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, support_count evidence accumulation, traverse_graph (outgoing recursive CTE))
