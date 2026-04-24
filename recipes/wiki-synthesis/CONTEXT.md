# CONTEXT.md — Wiki Synthesis

## Purpose

Synthesizes LLM-generated wiki articles from atomic thoughts stored in the Open Brain `thoughts` table. Provides two synthesis pipelines: a topic-scoped autobiographical narrative (grouped by life-year) and a per-Gmail-thread summarizer that produces `gmail_wiki` thoughts linked back to their source atoms via `thought_edges`.

## Responsibility Boundaries

- **Owns**: LLM synthesis logic, eligibility gating, prompt construction, output file management (markdown articles), and resume-safe JSONL state tracking for the Gmail backfill
- **Delegates to**: Supabase PostgREST for all data reads and writes; any OpenAI-compatible Chat Completions API for LLM calls; optional `upsert_thought` RPC (from the knowledge-graph recipe) for content-fingerprint-aware inserts
- **Does not handle**: Ingestion of raw Gmail data (that is `recipes/email-history-import/`), entity-level wiki pages (that is `recipes/entity-wiki/`), or vector search

## Key Concepts

**Synthesizer catalogue** (`SYNTHESIZERS` in `synthesize-wiki.mjs`): a named registry of synthesis strategies. `autobiography` is the only built-in; new topics are added by registering a new key with a `run({ args, api, env })` method.

**Life-date bucketing**: the autobiography synthesizer resolves a "life date" from a priority-ordered list of `metadata` JSONB keys (`event_at`, `life_date`, `source_date`, `captured_at`, `original_date`, `date`) before falling back to `created_at`. This allows thoughts captured later to be placed in the correct year.

**Thread eligibility**: the Gmail backfill gates synthesis on content weight, not message count — a thread must have `total_thread_word_count >= 500` AND (`distinct_messages >= 2` OR `atom_count >= 3`). Word-count accounting deduplicates by `gmail_id` when a thread contains both atomized and non-atomized messages.

**Resume-safe JSONL state**: `backfill-gmail-wikis.mjs` appends one JSON line per thread attempt to `./data/wiki-synthesis-state.jsonl`. Statuses are `ok`, `ok_partial_edges`, `failed`, and `skipped_ineligible`. The `--re-evaluate` flag overrides skip decisions; `--thread=` targets a single thread.

**wiki vs entity-wiki distinction**: this recipe synthesizes one article per topic or corpus slice using only the core `thoughts` table. `recipes/entity-wiki/` synthesizes one page per extracted entity and requires the entity-extraction schema.

## Non-Obvious Details

- The `captureWikiThought` function deletes existing `gmail_wiki` thoughts for a thread before inserting a new one, because the logical identity for a wiki is the `thread_id` (not a content fingerprint). This prevents duplicate wikis accumulating across re-runs.
- The `upsert_thought` RPC fallback only swallows HTTP 404 errors from `rpc/upsert_thought` (indicating the function is absent). Any other error — auth, permission, server error — is re-thrown to avoid silently bypassing whatever the RPC provides.
- Both scripts include prompt injection defenses: user-captured content is wrapped in `<entries>`/`<thread>` XML delimiters with explicit system-prompt instructions to treat the contents as quoted data only.
- The dashboard's `wiki/page.tsx` Server Action spawns `synthesize-wiki.mjs` as a child process using an explicit env-var allowlist to avoid leaking unrelated host secrets into the subprocess.
- Synthesized markdown articles include YAML frontmatter with `type`, `generated_at`, `source_count`, and optional `scope_year` and `dry_run` fields that the dashboard reads for display metadata.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue, Two-stage review pipeline)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, wiki vs entity-wiki distinction)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Resume-safe JSONL state)
- **[extensions/family-calendar](../../extensions/family-calendar/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, recurring vs. one-time activities in a unified table)
- **[extensions/home-maintenance](../../extensions/home-maintenance/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, frequency_days=NULL for one-time tasks, trigger-driven next_due recalculation)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Synthesizer catalogue)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Thread eligibility (content-weight gating))
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Noise filtering, Thread eligibility (content-weight gating))
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output modes (file / entity-metadata / thought), Resume-safe JSONL state)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Resume-safe JSONL state)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Thread eligibility (content-weight gating))
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Thread eligibility (content-weight gating), Transcript assembly)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Resume-safe JSONL state)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Thread eligibility (content-weight gating))
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Dynamic loop rescheduling, Resume-safe JSONL state)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility (content-weight gating), Topic shift detection)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Two-tier chunking (heading split + LLM fallback), wiki vs entity-wiki distinction)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Thread eligibility (content-weight gating))
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility (content-weight gating), Two-sheet import (Conversations + Memory))
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Skip rules, Thread eligibility (content-weight gating))
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Synthesizer catalogue)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Synthesizer catalogue)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Synthesizer catalogue)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Provides scripts/synthesize-wiki.mjs, scripts/backfill-gmail-wikis.mjs, dashboard-snippets/components/GenerateAutobiographyButton.tsx, ... consumed by this module
- **[recipes/wiki-synthesis/scripts](scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility (content-weight gating), Thread eligibility gate)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Thread eligibility (content-weight gating))
- **[schemas](../../schemas/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, Temporal validity and decay_weight)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Async queue with content-addressed re-queue, Resume-safe JSONL state)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, Temporal validity with NULL semantics, decay_weight)
- **[skills](../../skills/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Resume-safe JSONL state)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Synthesizer catalogue)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Resume-safe JSONL state)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Synthesizer catalogue)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread eligibility (content-weight gating))
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Thread eligibility (content-weight gating))
