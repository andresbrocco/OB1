# kubernetes-deployment

> Self-hosted Kubernetes deployment path for Open Brain — packages the MCP server as a Docker image with Deno runtime and deploys alongside a pgvector Postgres StatefulSet via Kubernetes manifests.

## Quick Reference

### Environment Variables

Secrets are injected via a Kubernetes `Secret` object (`openbrain-secret`). Copy `k8s/secrets.yml.example` to `k8s/secrets.yml`, fill in values, and apply before deploying.

| Variable | Description | Default in manifests | Required |
|----------|-------------|----------------------|----------|
| `DB_HOST` | Postgres host (loopback in pod) | `127.0.0.1` | Yes |
| `DB_PORT` | Postgres port | `5432` | Yes |
| `DB_NAME` | Postgres database name | `openbrain` | Yes |
| `DB_USER` | Postgres user | `postgres` | Yes |
| `DB_PASSWORD` | Postgres password (from secret `postgres-password`) | — | Yes |
| `MCP_ACCESS_KEY` | Bearer token clients use to authenticate (from secret `mcp-access-key`) | — | Yes |
| `EMBEDDING_API_BASE` | Base URL for the embedding model API | `https://openrouter.ai/api/v1` | Yes |
| `EMBEDDING_API_KEY` | API key for embedding provider (from secret `embedding-api-key`) | — | Yes |
| `EMBEDDING_MODEL` | Embedding model identifier | `openai/text-embedding-3-small` | Yes |
| `CHAT_API_BASE` | Base URL for the chat/completion model API | `https://openrouter.ai/api/v1` | Yes |
| `CHAT_API_KEY` | API key for chat provider (from secret `chat-api-key`) | — | Yes |
| `CHAT_MODEL` | Chat model identifier | `openai/gpt-4o-mini` | Yes |
| `PORT` | Port the MCP server listens on | `8000` | Yes |

> The variables `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are not used in this deployment path — the MCP server talks directly to the co-located Postgres container.

### Ports

| Port | Service | Protocol | Exposed as |
|------|---------|----------|------------|
| `8000` | `mcp-server` (MCP HTTP) | TCP | Kubernetes `ClusterIP` service; optionally via Ingress |
| `5432` | `db` (Postgres/pgvector) | TCP | Internal to pod only |

### Commands

```bash
# 1. Build the MCP server Docker image (run from this directory)
docker build -t openbrain-mcp-server .

# 2. Apply secrets (fill in k8s/secrets.yml from the example first)
cp k8s/secrets.yml.example k8s/secrets.yml
# ... edit k8s/secrets.yml with real values ...
kubectl apply -f k8s/secrets.yml

# 3. Deploy to Kubernetes
kubectl apply -f k8s/openbrain.yml

# 4. Local development (Deno, no Docker)
deno run --allow-net --allow-env --allow-read index.ts
```

### Configuration Files

| File | Purpose |
|------|---------|
| `Dockerfile` | Builds the MCP server image on `denoland/deno:2.3.3` |
| `deno.json` | Deno project manifest and dependency cache config |
| `k8s/openbrain.yml` | Full Kubernetes manifest: Namespace, ConfigMap, StatefulSet, Service, optional Ingress |
| `k8s/secrets.yml.example` | Template for the `openbrain-secret` Kubernetes Secret — copy and fill before applying |
| `k8s/init.sql` | Database initialisation SQL (creates `vector` extension, `thoughts` table, `match_thoughts` function) |

### Database Tables

| Table | Purpose |
|-------|---------|
| `thoughts` | Core memory store — `id`, `content`, `embedding vector(1536)`, `metadata JSONB`, `created_at` |

Initialised automatically on first pod start via the `init.sql` ConfigMap mounted at `/docker-entrypoint-initdb.d/init.sql`.

### Prerequisites

- Docker (any recent version)
- `kubectl` configured against a target cluster
- Deno 2.3.3 (for local development without Docker)
- A cluster node with sufficient memory (db container requests 256 Mi, limit 1 Gi; mcp-server requests 128 Mi, limit 512 Mi)
- Writable host path `/var/openbrain/db` on the target node (used for Postgres data persistence)

## Common Tasks

### Deploy from scratch

```bash
# Build image
docker build -t openbrain-mcp-server .

# Prepare secrets (edit values in the copied file)
cp k8s/secrets.yml.example k8s/secrets.yml

# Apply secrets, then the full stack
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/openbrain.yml

# Confirm pods are ready
kubectl -n openbrain get pods
```

### Check service health

```bash
# Watch pod readiness (both containers must reach 2/2 Running)
kubectl -n openbrain get pods -w

# View MCP server logs
kubectl -n openbrain logs statefulset/openbrain -c mcp-server

# View Postgres logs
kubectl -n openbrain logs statefulset/openbrain -c db
```

### Expose the MCP server externally

The `openbrain` Service is a `ClusterIP` by default. To make it reachable from Claude Desktop, uncomment and configure the Ingress block at the bottom of `k8s/openbrain.yml`, then re-apply:

```bash
# After editing the Ingress section with your domain and TLS secret:
kubectl apply -f k8s/openbrain.yml

# Add the MCP server as a custom connector in Claude Desktop:
# Settings → Connectors → Add custom connector → paste https://brain.yourdomain.com
```

### Tear down

```bash
# Remove all Open Brain resources (keeps host path data intact)
kubectl delete -f k8s/openbrain.yml
kubectl delete -f k8s/secrets.yml

# Delete the namespace entirely
kubectl delete namespace openbrain
```

### Update the MCP server image

```bash
docker build -t openbrain-mcp-server .
# If using a registry, push and update the image tag in k8s/openbrain.yml, then:
kubectl rollout restart statefulset/openbrain -n openbrain
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `mcp-server` container in `CrashLoopBackOff` | DB not ready yet when server starts | Wait — the readiness probe retries every 10 s; the server will restart until Postgres accepts connections |
| `ImagePullBackOff` on `openbrain-mcp-server:latest` | Image not present on the node | Run `docker build -t openbrain-mcp-server .` on the node, or push to a registry and update `imagePullPolicy` |
| Pod stuck at `0/1` or `1/2` ready | Postgres init taking longer than 10 s | Increase `initialDelaySeconds` in the readiness probe in `k8s/openbrain.yml` |
| `permission denied` on `/var/openbrain/db` | Host path doesn't exist or wrong ownership | `mkdir -p /var/openbrain/db` on the node; the manifest uses `DirectoryOrCreate` but the path must be writable by the postgres UID |
| MCP client gets `401 Unauthorized` | Wrong or missing `MCP_ACCESS_KEY` | Verify the secret value matches what the client sends; re-apply `k8s/secrets.yml` and restart the pod |
| Embedding or chat calls fail | `EMBEDDING_API_KEY` / `CHAT_API_KEY` incorrect or wrong base URL | Check secret values in `k8s/secrets.yml`; confirm the model identifiers match your provider's naming |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context
- [k8s/CONTEXT.md](k8s/CONTEXT.md) — Kubernetes manifest details
- [../README.md](../README.md) — Integrations overview
