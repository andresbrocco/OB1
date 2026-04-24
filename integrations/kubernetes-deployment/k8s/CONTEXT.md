# CONTEXT.md — k8s

## Purpose

Kubernetes manifests and database initialization for self-hosting Open Brain outside of Supabase. Defines the full deployment surface: namespace, secrets, ConfigMap-embedded SQL schema, StatefulSet, and ClusterIP Service.

## Responsibility Boundaries

- **Owns**: All Kubernetes resource definitions needed to run Open Brain (PostgreSQL+pgvector and MCP server) on a self-managed cluster
- **Delegates to**: The parent `integrations/kubernetes-deployment/` directory for operator-facing documentation and the custom `openbrain-mcp-server` container image (built separately, not defined here)
- **Does not handle**: Ingress/TLS termination (commented-out template only), persistent volume provisioning beyond a `hostPath`, or image builds

## Key Concepts

- **Co-located pod pattern**: The PostgreSQL (`db`) and MCP server (`mcp-server`) containers run in the same StatefulSet pod and communicate via `127.0.0.1`, avoiding a separate Service for intra-pod DB access.
- **ConfigMap-embedded SQL**: `init.sql` is duplicated — once as a standalone file for direct `psql` use and once inlined into the `openbrain-init-sql` ConfigMap. The ConfigMap copy is what actually runs at container init via `/docker-entrypoint-initdb.d/`.
- **Supabase schema parity**: `init.sql` is a self-hosted equivalent of the Supabase-managed schema. It defines the `thoughts` table and the `match_thoughts` PL/pgSQL function, which mirrors the Supabase RPC surface used by all OB1 MCP tools.

## Non-Obvious Details

- The `hostPath` volume at `/var/openbrain/db` requires that path to exist and be writable on the node before the pod starts. There is no dynamic PVC; this is intentional for single-node simplicity but breaks multi-node or cloud deployments.
- `secrets.yml.example` must be copied to `secrets.yml`, filled in, and applied with `kubectl apply` before the StatefulSet can start. The example file is committed; the real file must never be committed.
- The StatefulSet uses `replicas: 1` and is not designed to scale horizontally — pgvector does not support multi-primary replication out of the box.
- The Ingress block is fully commented out. TLS and external hostname access require uncommenting, updating `brain.yourdomain.com`, and ensuring a compatible ingress controller (nginx or traefik) is installed.

## Related Modules

- **[integrations](../../CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Kubernetes self-hosted MCP server, Supabase schema parity, hostPath volume)
- **[integrations/kubernetes-deployment](../CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Supabase replacement pattern, Supabase schema parity, hostPath volume)
- **[recipes/entity-wiki](../../../recipes/entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Semantic expansion, match_thoughts RPC equivalent)
- **[recipes/live-retrieval](../../../recipes/live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), match_thoughts RPC equivalent)
- **[recipes/local-ollama-embeddings](../../../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, match_thoughts RPC equivalent)
- **[recipes/repo-learning-coach/src](../../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Co-located pod pattern, ConfigMap-embedded SQL, Supabase schema parity, hostPath volume)
- **[recipes/repo-learning-coach/src/lib](../../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Co-located pod pattern, ConfigMap-embedded SQL, Supabase schema parity, hostPath volume)
- **[recipes/vercel-neon-telegram/src/lib](../../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (match_thoughts, match_thoughts RPC equivalent)
- **[schemas](../../../schemas/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Two-phase full-text search (GIN tsvector + ILIKE fallback), match_thoughts RPC equivalent)
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Supabase schema parity, hostPath volume, idempotent schema migration)
- **[server](../../../server/CONTEXT.md)** — Shares Vector Search and Retrieval domain (match_thoughts RPC (pgvector similarity search), match_thoughts RPC equivalent)
- **[skills/weekly-signal-diff](../../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Starter universe bootstrap, Supabase schema parity, hostPath volume)
