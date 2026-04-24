# CONTEXT.md — Local Ollama Embeddings

## Purpose

Provides a CLI tool (`embed-local.py`) for ingesting thoughts into the Open Brain Supabase database using locally-run Ollama embedding models instead of cloud-based embedding APIs. Designed for offline or privacy-sensitive workflows where no OpenRouter or external API key should be used for the embedding step.

## Responsibility Boundaries

- **Owns**: Embedding generation via the local Ollama `/api/embed` endpoint, input parsing (stdin, positional arguments, `.txt` files, `.jsonl` files), and direct REST insertion into the Supabase `thoughts` table.
- **Delegates to**: Ollama (embedding inference), Supabase REST API (persistence).
- **Does not handle**: Semantic search, retrieval, deduplication, or any post-ingestion processing.

## Key Concepts

- **Local embedding**: Embeddings are computed by a locally-running Ollama instance rather than a remote API, eliminating cloud cost and data exposure for the embedding step. Supabase writes still require network access.
- **Supported input modes**: Plain text (one thought per line), JSONL (objects with a required `content` key plus optional `source` and `metadata` fields), stdin piping, or direct positional arguments.
- **Model dimension mismatch**: The default Open Brain schema stores embeddings as `vector(1536)`. Most bundled Ollama models produce different dimensions (`nomic-embed-text` → 768, `mxbai-embed-large` → 1024). The script warns at runtime when dimensions differ and prints the `ALTER TABLE` statement needed to resize the column.

## Non-Obvious Details

- Input text is silently truncated to 8,000 characters before being sent to Ollama. There is no warning emitted for truncated input.
- The script uses a hard 0.1-second sleep between insertions as a gentle rate limit against Supabase; there is no configurable rate limit flag.
- Connectivity to Ollama is checked via `GET /api/tags` at startup; failure exits immediately rather than falling back gracefully.
- `--batch-size` is accepted as a CLI argument but the embedding loop always processes one thought at a time — the parameter is parsed but not used to batch requests to Ollama.
- The service role key is used directly for Supabase writes (bypassing RLS). This is intentional for a local ingestion tool but means `.env` values must not be committed or shared.

## Related Modules

- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), Embedding dimension mismatch, Local embedding via Ollama, pgvector cosine distance via raw SQL)
- **[integrations/kubernetes-deployment/k8s](../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, match_thoughts RPC equivalent)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Import and Export Pipelines domain (Multi-format input (stdin, positional args, .txt, .jsonl), Portable Bundle)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Body cleaning pipeline, Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, Semantic expansion)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (MongoDB-style date parsing, Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Meta latin1/UTF-8 encoding repair, Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Hit threshold (score > 0.6), Local embedding via Ollama, Retrieval log, Session cap (max 3 retrievals))
- **[recipes/repo-learning-coach/src](../repo-learning-coach/src/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, RelatedThoughts)
- **[recipes/repo-learning-coach/src/lib](../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, RelatedThoughtSummary)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, match_thoughts)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Multi-format input (stdin, positional args, .txt, .jsonl), Tweet batching, Twitter JS export format)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, two-phase GIN+ILIKE full-text search)
- **[server](../../server/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, match_thoughts RPC (pgvector similarity search))
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Multi-format input (stdin, positional args, .txt, .jsonl))
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Multi-format input (stdin, positional args, .txt, .jsonl), export bundles)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Live search upgrade, Local embedding via Ollama)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), Multi-format input (stdin, positional args, .txt, .jsonl))
