# CONTEXT.md — ChatGPT Conversation Import

## Purpose

Transforms a ChatGPT data export (zip or extracted directory) into structured knowledge thoughts stored in Open Brain. The pipeline parses the export's conversation tree format, filters low-signal conversations, runs LLM-based knowledge extraction to produce 2–5 typed thoughts per conversation, and inserts them into Supabase with embeddings.

## Responsibility Boundaries

- **Owns**: Export file parsing, conversation branch resolution, content-type dispatch, session splitting, signal-based filtering, LLM extraction prompt construction, deduplication via sync log, and Supabase ingestion.
- **Delegates to**: OpenRouter (or Ollama) for LLM knowledge extraction and embedding generation; Supabase for persistence and vector search.
- **Does not handle**: Ongoing capture or real-time sync; this is a one-time (or incremental re-run) batch import tool.

## Key Concepts

**Conversation tree / branch resolution** — ChatGPT exports store messages as a mapping of nodes with parent pointers, not a flat list. `resolve_canonical_path` in `chatgpt_parser.py` reconstructs the linear path the user actually followed: it prefers `current_node` (the leaf of the last active branch), falling back to a largest-subtree heuristic when `current_node` is absent.

**Content-type dispatch** — ChatGPT messages carry a `content_type` field (at least 14 variants observed). The parser maintains an explicit skip-set (`SKIP_CONTENT_TYPES`) for model-internal types (chain-of-thought, browser screenshots, etc.) and applies per-type extraction logic for the rest (e.g., stripping citation markers from `tether_browsing_display`, extracting audio transcriptions from `multimodal_text`).

**Signal-based filtering** — Rather than regex title matching, conversations are scored by message count and word count: fewer than 2 messages are always skipped; 10+ messages are always processed; the borderline range (2–9) applies word-count and untitled-short heuristics, then defers to the LLM's own `skip_reason` field for ambiguous cases.

**Session splitting** — Long multi-day conversations are split on 4-hour timestamp gaps before being sent to the LLM, to avoid exceeding the ~100k-token context window. Individual sessions that still exceed the limit are head+tail truncated (~3 000 + 1 000 tokens).

**Pyramid summaries** — The optional `--store-conversations` flag generates five summary lengths (8w, 16w, 32w, 64w, 128w) per conversation and stores them in the `chatgpt_conversations` table (defined in `schema.sql`). The 128-word summary is embedded for conversation-level semantic search, independent of individual thought embeddings.

**Sync log** — A local JSON file (`chatgpt_sync.json`) tracks ingested conversation hashes and their `update_time`. Re-runs skip unchanged conversations and re-process ones where `update_time` has advanced (new messages appended).

## Non-Obvious Details

- The export parser handles two OpenAI export formats: single `conversations.json` (older exports) and the chunked `conversations-000.json … conversations-NNN.json` format used for large accounts. Both zip and pre-extracted directories are accepted.
- `conversation_hash` is derived from `title + create_time`, not the ChatGPT conversation ID, because the ID is not always present or stable across export versions.
- The sync log entry format changed between versions: old entries are bare strings; new entries are dicts with `update_time`. `should_skip` handles both formats transparently.
- `schema.sql` must be run manually in Supabase before using `--store-conversations`; it is not applied automatically by the importer.
- The `--ingest-endpoint` flag routes ingestion through a custom HTTP endpoint (e.g., a Supabase Edge Function) instead of direct Supabase client inserts, enabling use behind an API gateway.
- Environment variable `USER_ID` is required for `--store-conversations` in multi-tenant setups; without it, `user_id` is set via `auth.uid()`, which is `NULL` under `service_role` authentication.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Sync log)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, Sync log)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Pyramid summaries)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Sync log, re-extraction idempotency)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Sync log, Two-layer dedup)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Pyramid summaries, Slug collision resolution)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Sync log)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Sync log)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Conversation tree / branch resolution, Session splitting, Transcript assembly)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Manifest file, Pyramid summaries)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Sync log)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Sync log)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Sync log)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Topic shift detection)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Sync log)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Speaker Consolidation, Thread)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Two-sheet import (Conversations + Memory))
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal-based filtering, Skip rules)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Schema-aware routing)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, LLM classification prompt with importance/confidence calibration)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Sync log)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Compile manifest, Compiled wiki, Pyramid summaries)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Thread eligibility gate)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session Versioning, Session splitting)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Sync log)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Sync log, idempotent schema migration)
- **[skills](../../skills/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Output Contract, Pyramid summaries)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, Sync log)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Deterministic-first policy)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (.ob1 output directory, Pyramid summaries)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Mode classification)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Speaker Consolidation)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Signal-based filtering)
