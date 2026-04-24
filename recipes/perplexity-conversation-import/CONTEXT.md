# CONTEXT.md — Perplexity Conversation Import

## Purpose

A CLI recipe that imports Perplexity AI search history into Open Brain. It reads a Perplexity data export (.xlsx file), processes two distinct data types (conversation Q&A pairs and structured memory entries), and inserts them into the `thoughts` table with vector embeddings.

## Responsibility Boundaries

- **Owns**: Parsing the Perplexity .xlsx export format, LLM-based summarization of conversations into distilled thoughts, JSON profile flattening, deduplication via a local sync log, and direct Supabase REST ingestion with embeddings
- **Delegates to**: OpenRouter (summarization via gpt-4o-mini and embeddings via text-embedding-3-small) or a local Ollama instance for the LLM work; Supabase REST API for persistence
- **Does not handle**: Fetching data directly from Perplexity (requires a manual export), ongoing sync or incremental polling, duplicate detection at the database level

## Key Concepts

- **Two-sheet import**: The .xlsx export contains a "Conversations" sheet (Q&A pairs with UUID, title, and an OUTPUT_STR JSON blob) and a "Memory" sheet (key-value pairs that Perplexity inferred about the user). These are processed through separate pipelines.
- **JSON profile rows**: Some Memory rows have no MEMORY_KEY and carry a structured JSON object in MEMORY_VALUE representing Perplexity's inferred user profile. These are detected by `is_json_profile_row` and flattened section-by-section (demographics, interests, work_and_education, etc.) rather than treated as plain key-value memories.
- **Selective summarization**: Conversations are not stored verbatim. An LLM with a deliberately conservative prompt distills each Q&A into 0–3 standalone first-person thoughts. The prompt explicitly instructs skipping trivial lookups, so many conversations produce no output and are marked done without ingestion.
- **Local sync log**: A `perplexity-sync-log.json` file tracks ingested item hashes (truncated SHA256) to prevent re-importing items across runs. Deduplication keys differ by item type: UUID-based for conversations, key+timestamp for memory rows, content-hash for JSON profile rows.
- **OUTPUT_STR parsing**: Conversation answer text is embedded as a JSON blob in the OUTPUT_STR column. `_parse_output_str` unwraps it to extract the `answer` field.

## Non-Obvious Details

- Memory rows flagged `IS_DELETED` or `IS_FORGOTTEN` are silently skipped; the boolean normalization handles string representations ("true", "1", "yes").
- A conversation that produces no thoughts after summarization is still recorded in the sync log so it is not re-processed on future runs.
- The sync log is only updated after all thoughts for an item succeed (`all_ok`). A partial failure leaves the item unlogged, so it will be retried on the next run.
- Ingested thought content is prefixed with a source tag (e.g., `[Perplexity: {title} | {date}]`) to make source attribution visible in retrieval results without relying solely on metadata.
- The cost estimator printed at the end uses hardcoded token averages (800 input / 200 output per conversation, 100 tokens per thought for embeddings) and current OpenRouter rates — useful for estimating but not a live measurement.
- `--dry-run` with `--model openrouter` will call the LLM to generate summaries but will not ingest, giving a preview of what would be stored. Without an `OPENROUTER_API_KEY`, summarization is skipped entirely even in dry-run mode.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Local sync log deduplication)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, Local sync log deduplication)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, details JSONB freeform metadata field)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, JSONB ingredient and shopping item storage)
- **[integrations](../../integrations/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Shared config with sensitivity tiers)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Local sync log deduplication, re-extraction idempotency)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Selective LLM summarization)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Two-sheet import (Conversations + Memory))
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Local sync log deduplication, Sync log, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Local sync log deduplication)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Local sync log deduplication)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Transcript assembly, Two-sheet import (Conversations + Memory))
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Local sync log deduplication)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Local sync log deduplication)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Local sync log deduplication)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Topic shift detection, Two-sheet import (Conversations + Memory))
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Local sync log deduplication)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Two-sheet import (Conversations + Memory))
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Selective LLM summarization)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Type backfill (metadata.type promotion))
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Local sync log deduplication)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, ThoughtMetadata)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, ThoughtMetadata)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility (content-weight gating), Two-sheet import (Conversations + Memory))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility gate, Two-sheet import (Conversations + Memory))
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Two-sheet import (Conversations + Memory))
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Local sync log deduplication)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Local sync log deduplication, idempotent schema migration)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Local sync log deduplication, Open Brain deduplication workflow)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Model shape)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Selective LLM summarization)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Product shape)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Two-sheet import (Conversations + Memory))
