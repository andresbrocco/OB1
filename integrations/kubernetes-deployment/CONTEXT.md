# CONTEXT.md — Kubernetes Deployment

## Purpose

A self-hosted variant of the Open Brain MCP server that replaces Supabase with a direct PostgreSQL + pgvector connection. It exposes the same four MCP tools (`capture_thought`, `search_thoughts`, `list_thoughts`, `thought_stats`) via a Hono HTTP server running inside a Docker container, intended for deployment on Kubernetes infrastructure.

## Responsibility Boundaries

- **Owns**: The full MCP server lifecycle — tool registration, HTTP transport, connection pooling, embedding generation, and metadata extraction — without any Supabase dependency.
- **Delegates to**: An external OpenAI-compatible embedding API (e.g., OpenRouter) for vector generation, and an external OpenAI-compatible chat API for metadata extraction from captured thoughts.
- **Does not handle**: Kubernetes manifests, ingress configuration, database provisioning, or pgvector extension setup. Those are left to the operator.

## Key Concepts

- **Supabase replacement pattern**: The canonical OB1 server uses Supabase client calls and `supabase.rpc()` for vector search. This integration substitutes all of those with raw SQL against a plain PostgreSQL pool, using the `<=>` pgvector cosine distance operator directly.
- **Dual API configuration**: Embedding and chat completion endpoints are independently configurable (`EMBEDDING_API_BASE`/`EMBEDDING_API_KEY` vs `CHAT_API_BASE`/`CHAT_API_KEY`). The chat variables fall back to the embedding values, allowing a single OpenRouter key to serve both roles or separating them for cost or latency reasons.
- **Authentication**: Access is gated by `MCP_ACCESS_KEY`, checked via the `x-brain-key` header or a `key` query parameter on every request. There is no per-user identity; all requests share one key.

## Non-Obvious Details

- The repo-wide rule is that MCP servers must be remote Supabase Edge Functions. This integration is an explicitly supported exception for operators who want fully self-managed infrastructure. It runs as a long-lived Deno process (not a serverless function) and must be exposed via a public HTTPS endpoint to work with Claude Desktop's custom connector UI.
- Vector embeddings are serialized as a plain string `[f1,f2,...]` and cast to `vector` at the SQL layer (`$2::vector`). The pgvector extension must already be enabled and the `embedding` column must be of type `vector` in the target database.
- The `thought_stats` tool fetches all rows (`SELECT metadata, created_at FROM thoughts`) and aggregates in application memory. This will degrade at large thought counts; it is not paginated.
- The Dockerfile runs as the `deno` non-root user and grants only `--allow-net`, `--allow-env`, and `--allow-read` permissions — no filesystem write access.

## Related Modules

- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, SSR auth)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Restricted content unlock, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Restricted content passphrase gating)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, MCP_ACCESS_KEY authentication)
- **[docs](../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Query-parameter auth pattern)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, MCP_ACCESS_KEY pre-shared key authentication)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, MCP_ACCESS_KEY authentication)
- **[integrations](../CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Kubernetes self-hosted MCP server, Supabase replacement pattern)
- **[integrations/kubernetes-deployment/k8s](k8s/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Supabase replacement pattern, Supabase schema parity, hostPath volume)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), Semantic expansion, pgvector cosine distance via raw SQL)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), pgvector cosine distance via raw SQL)
- **[recipes/local-ollama-embeddings](../../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), Embedding dimension mismatch, Local embedding via Ollama, pgvector cosine distance via raw SQL)
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Supabase replacement pattern)
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Supabase replacement pattern)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Sensitivity tiers (standard/personal/restricted))
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, MCP_ACCESS_KEY authentication, Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), match_thoughts, pgvector cosine distance via raw SQL)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, sensitivity_tier)
- **[server](../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, x-brain-key access key auth)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Starter universe bootstrap, Supabase replacement pattern)
