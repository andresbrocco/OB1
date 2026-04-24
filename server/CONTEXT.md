# CONTEXT.md — Server

## Purpose

The core Open Brain MCP server: a Deno/Supabase Edge Function that exposes the `thoughts` database over the Model Context Protocol. It handles thought capture (with automatic embedding generation and metadata extraction), semantic search, chronological listing, and aggregate statistics — all accessible to any MCP-compatible AI client.

## Responsibility Boundaries

- **Owns**: MCP tool definitions, HTTP transport wiring, access key authentication, embedding generation via OpenRouter, metadata extraction via LLM, and thought persistence via the `upsert_thought` RPC and `thoughts` table.
- **Delegates to**: OpenRouter (embeddings via `text-embedding-3-small`, metadata extraction via `gpt-4o-mini`), Supabase (`match_thoughts` RPC for vector search, `upsert_thought` RPC for deduplication-aware inserts).
- **Does not handle**: Schema migrations, user management, dashboard rendering, or webhook ingestion — those live in `schemas/`, `extensions/`, and `integrations/`.

## Key Concepts

- **`match_thoughts` RPC**: A Supabase stored procedure that performs pgvector cosine similarity search. The `search_thoughts` tool calls this rather than doing similarity math in application code.
- **`upsert_thought` RPC**: A stored procedure that handles content-level deduplication on insert; the server calls it first, then updates the embedding in a second round-trip because the RPC does not accept the embedding vector directly.
- **Two-step capture**: `capture_thought` fires embedding generation and metadata extraction in parallel (`Promise.all`), then upserts content, then patches the embedding in a separate `UPDATE`. This split exists because the upsert RPC predates vector storage support.
- **`x-brain-key` auth**: Access is gated by a single shared secret passed as either the `x-brain-key` header or a `key` query parameter. The dual-mode support is intentional for clients (e.g., Claude Desktop custom connectors) that cannot set arbitrary headers.

## Non-Obvious Details

- **Accept header patch**: Claude Desktop's custom connector does not send `Accept: text/event-stream`, which `StreamableHTTPTransport` requires. The server detects the missing header and reconstructs the raw request with the header injected before handing off to the transport. The `@ts-ignore` on `duplex: "half"` is required for streaming body support in Deno and is load-bearing.
- **Deployment target**: This is a Supabase Edge Function (Deno runtime), not a standalone Node.js server. `Deno.serve` is the entry point. It must be deployed via `supabase functions deploy`, not run locally with Node.
- **Environment variables required at runtime**: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `OPENROUTER_API_KEY`, `MCP_ACCESS_KEY`. All four must be set as Supabase secrets; the server will throw at startup if any are absent (non-null assertion via `!`).
- **CORS scope**: The wildcard `Access-Control-Allow-Origin: *` is intentional to support browser-based and Electron-based clients (claude.ai, Claude Desktop). The access key provides the actual security boundary.

## Related Modules

- **[dashboards](../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, sensitivity_tier restricted content gating, x-brain-key access key auth)
- **[dashboards/open-brain-dashboard](../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (SSR auth, x-brain-key access key auth)
- **[dashboards/open-brain-dashboard-next](../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard, x-brain-key access key auth)
- **[dashboards/open-brain-dashboard-next/app/api](../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content unlock, Session-scoped API key forwarding, x-brain-key access key auth)
- **[dashboards/open-brain-dashboard-next/components](../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Restricted content passphrase gating, x-brain-key access key auth)
- **[dashboards/open-brain-dashboard-next/lib](../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key, x-brain-key access key auth)
- **[dashboards/open-brain-dashboard/src](../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP credential server-vs-public pattern, MCP tool registration, StreamableHTTPTransport)
- **[dashboards/open-brain-dashboard/src/lib](../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP text-response parsing, MCP tool registration, StreamableHTTPTransport)
- **[dashboards/open-brain-dashboard/src/routes](../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, x-brain-key access key auth)
- **[docs](../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, x-brain-key access key auth)
- **[extensions](../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP tool registration, Per-request MCP server instantiation, StreamableHTTPTransport)
- **[extensions/family-calendar](../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, Claude Desktop Accept-header patch, MCP tool registration, StreamableHTTPTransport)
- **[extensions/household-knowledge](../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, x-brain-key access key auth)
- **[extensions/meal-planning](../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, x-brain-key access key auth)
- **[extensions/professional-crm](../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP tool registration, Stateless MCP transport, StreamableHTTPTransport)
- **[integrations](../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, Kubernetes self-hosted MCP server, MCP tool registration, StreamableHTTPTransport)
- **[integrations/entity-extraction-worker/_shared](../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, Two-step capture (upsert + embedding patch), prepareThoughtPayload, upsert_thought RPC (deduplication-aware insert))
- **[integrations/kubernetes-deployment](../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, x-brain-key access key auth)
- **[integrations/kubernetes-deployment/k8s](../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Vector Search and Retrieval domain (match_thoughts RPC (pgvector similarity search), match_thoughts RPC equivalent)
- **[recipes/email-history-import](../recipes/email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Ingestion modes, Two-step capture (upsert + embedding patch), upsert_thought RPC (deduplication-aware insert))
- **[recipes/entity-wiki](../recipes/entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Semantic expansion, match_thoughts RPC (pgvector similarity search))
- **[recipes/google-activity-import](../recipes/google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Thought prefix format on insert, Two-step capture (upsert + embedding patch), upsert_thought RPC (deduplication-aware insert))
- **[recipes/instagram-import](../recipes/instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Two-step capture (upsert + embedding patch), upsert_thought RPC, upsert_thought RPC (deduplication-aware insert))
- **[recipes/live-retrieval](../recipes/live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), match_thoughts RPC (pgvector similarity search))
- **[recipes/local-ollama-embeddings](../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, match_thoughts RPC (pgvector similarity search))
- **[recipes/obsidian-vault-import](../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Two-step capture (upsert + embedding patch), upsert_thought RPC (deduplication-aware insert))
- **[recipes/repo-learning-coach/src](../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughts, match_thoughts RPC (pgvector similarity search))
- **[recipes/repo-learning-coach/src/lib](../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughtSummary, match_thoughts RPC (pgvector similarity search))
- **[recipes/thought-enrichment](../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Sensitivity tiers (standard/personal/restricted), x-brain-key access key auth)
- **[recipes/vercel-neon-telegram](../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Accept header patch for Claude Desktop compatibility, MCP tool registration, Stateless MCP transport, StreamableHTTPTransport)
- **[recipes/vercel-neon-telegram/src](../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (timingSafeEqual auth, x-brain-key access key auth)
- **[recipes/vercel-neon-telegram/src/app/api](../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Telegram webhook secret authentication, x-brain-key access key auth)
- **[recipes/vercel-neon-telegram/src/lib](../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Two-step capture (upsert + embedding patch), captureThought pipeline, upsert_thought RPC (deduplication-aware insert))
- **[schemas](../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier access filtering, x-brain-key access key auth)
- **[schemas/enhanced-thoughts](../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (sensitivity_tier, x-brain-key access key auth)
- **[skills/weekly-signal-diff](../skills/weekly-signal-diff/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, match_thoughts RPC (pgvector similarity search))
