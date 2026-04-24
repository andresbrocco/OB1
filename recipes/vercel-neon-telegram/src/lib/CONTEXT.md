# CONTEXT.md — lib

## Purpose

Core shared library for the vercel-neon-telegram recipe. Provides the full thought-capture pipeline: authentication, rate limiting, AI enrichment (embedding + metadata extraction), and database persistence against a Neon (serverless Postgres + pgvector) instance.

## Responsibility Boundaries

- **Owns**: Thought ingestion pipeline, access-key authentication, in-memory rate limiting, OpenAI embedding and metadata extraction, Neon DB read/write operations, shared TypeScript types and Zod schemas
- **Delegates to**: Vercel API route handlers (HTTP request parsing, response formatting), OpenAI via Vercel AI SDK (model calls), Neon serverless driver (connection pooling)
- **Does not handle**: Telegram bot protocol, webhook verification, environment configuration loading beyond reading `process.env`

## Key Concepts

- **Thought**: The core domain object — a piece of content stored with a pgvector embedding, structured `ThoughtMetadata`, and a `source` tag identifying the capture channel (e.g., `"telegram"`).
- **ThoughtMetadata**: AI-extracted structured fields (`people`, `action_items`, `dates_mentioned`, `topics`, `type`). Classified into seven enumerated thought types: `observation`, `task`, `idea`, `reference`, `person_note`, `decision`, `meeting_note`.
- **Capture pipeline**: `captureThought` in `capture.ts` fans out embedding generation and metadata extraction in parallel (`Promise.all`), then persists via `insertThought`.
- **match_thoughts**: A Postgres function (defined outside this lib) used for vector similarity search. `searchThoughts` calls it via a tagged template literal; the similarity threshold defaults to `0.7`.

## Non-Obvious Details

- **Rate limiter resets on cold start.** The in-memory `requests` array in `rate-limit.ts` is module-level state. In Vercel's serverless runtime each cold start produces a fresh instance, so the 30-requests-per-minute window is per-instance, not global. The module comment acknowledges this is intentional for personal use.
- **Mixed query styles in `db.ts`.** `insertThought` and `searchThoughts` use Neon's tagged template literal API (`sql\`...\``), but `listThoughts` falls back to `sql.query(query, params)` because it builds a dynamic `WHERE` clause at runtime. These two styles behave differently in how parameters are serialized.
- **Auth supports three key-extraction strategies** (`x-brain-key` header, `Authorization: Bearer`, `?key=` query param) to accommodate clients that cannot set custom headers (e.g., ChatGPT actions). Key comparison uses `timingSafeEqual` to prevent timing attacks.
- **Embedding vectors are serialized to JSON strings** before being passed to Neon (`JSON.stringify(embedding)`) and cast via `::vector` in the SQL. This is required by the Neon serverless driver's handling of pgvector parameters.

## Related Modules

- **[dashboards](../../../../dashboards/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Smart ingest auto-routing heuristic, captureThought pipeline)
- **[dashboards/open-brain-dashboard](../../../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType)
- **[dashboards/open-brain-dashboard-next](../../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Auto-routing heuristic (shouldExtract), captureThought pipeline)
- **[dashboards/open-brain-dashboard-next/components](../../../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Dry-run two-phase ingestion, Ingestion modes (auto/single/extract), captureThought pipeline)
- **[dashboards/open-brain-dashboard-next/lib](../../../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, captureThought pipeline)
- **[dashboards/open-brain-dashboard/src](../../../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Thought-type color tokens, ThoughtType)
- **[dashboards/open-brain-dashboard/src/lib](../../../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType)
- **[extensions/household-knowledge](../../../../extensions/household-knowledge/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (ThoughtMetadata, details JSONB freeform metadata field)
- **[extensions/meal-planning](../../../../extensions/meal-planning/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, ThoughtMetadata)
- **[integrations](../../../../integrations/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Shared config with sensitivity tiers, ThoughtMetadata)
- **[integrations/entity-extraction-worker/_shared](../../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Structured capture format, captureThought pipeline, prepareThoughtPayload)
- **[integrations/kubernetes-deployment](../../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), match_thoughts, pgvector cosine distance via raw SQL)
- **[integrations/kubernetes-deployment/k8s](../../../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Vector Search and Retrieval domain (match_thoughts, match_thoughts RPC equivalent)
- **[recipes/email-history-import](../../../email-history-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Ingestion modes, captureThought pipeline)
- **[recipes/entity-wiki](../../../entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Semantic expansion, match_thoughts)
- **[recipes/google-activity-import](../../../google-activity-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Thought prefix format on insert, captureThought pipeline)
- **[recipes/instagram-import](../../../instagram-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (captureThought pipeline, upsert_thought RPC)
- **[recipes/live-retrieval](../../../live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), match_thoughts)
- **[recipes/local-ollama-embeddings](../../../local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, match_thoughts)
- **[recipes/obsidian-vault-import](../../../obsidian-vault-import/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, captureThought pipeline)
- **[recipes/panning-for-gold](../../../panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, ThoughtType)
- **[recipes/perplexity-conversation-import](../../../perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, ThoughtMetadata)
- **[recipes/repo-learning-coach/server](../../../repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), ThoughtType)
- **[recipes/repo-learning-coach/src](../../../repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/repo-learning-coach/src/lib](../../../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, ThoughtType)
- **[recipes/thought-enrichment](../../../thought-enrichment/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (ThoughtMetadata, Type backfill (metadata.type promotion))
- **[recipes/vercel-neon-telegram](../../CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Parallel capture pipeline, captureThought pipeline)
- **[recipes/vercel-neon-telegram/src](../CONTEXT.md)** — Shares JSONB and Schema Metadata domain (ThoughtMetadata)
- **[schemas](../../../../schemas/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Two-phase full-text search (GIN tsvector + ILIKE fallback), match_thoughts)
- **[schemas/enhanced-thoughts](../../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Vector Search and Retrieval domain (match_thoughts, two-phase GIN+ILIKE full-text search)
- **[server](../../../../server/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Two-step capture (upsert + embedding patch), captureThought pipeline, upsert_thought RPC (deduplication-aware insert))
- **[skills/financial-model-review](../../../../skills/financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, ThoughtMetadata)
- **[skills/n-agentic-harnesses](../../../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, ThoughtMetadata)
- **[skills/panning-for-gold](../../../../skills/panning-for-gold/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../../../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, match_thoughts)
