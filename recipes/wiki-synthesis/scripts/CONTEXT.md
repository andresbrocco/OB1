# CONTEXT.md — wiki-synthesis/scripts

## Purpose

Executable Node.js scripts that synthesize structured wiki articles from atomic thoughts stored in Open Brain. Two complementary workflows are provided: a general topic-scoped synthesizer that can produce any corpus-slice article (e.g., autobiographical narrative), and a Gmail-specific backfill synthesizer that groups imported email threads into wiki-style summaries linked back to their source atoms.

## Responsibility Boundaries

- **Owns**: Thought retrieval from Supabase PostgREST, LLM prompt construction and call, wiki article assembly, output file writing, and resume-safe state tracking for the Gmail backfill.
- **Delegates to**: Supabase (`thoughts`, `thought_edges`, `upsert_thought` RPC) for persistence; any OpenAI-compatible Chat Completions endpoint for synthesis.
- **Does not handle**: Embedding, vector search, entity extraction, or ingestion of source data (Gmail import is assumed done separately by `recipes/email-history-import/`).

## Key Concepts

- **Synthesizer catalogue** (`SYNTHESIZERS` in `synthesize-wiki.mjs`): a plugin map where each key is a named synthesizer slug (e.g., `autobiography`). New synthesizers are added by extending this object — they receive `{ args, api, env }` and are responsible for fetching, prompting, and writing output.
- **Life-date bucketing**: The autobiography synthesizer prioritizes a set of metadata jsonb keys (`event_at`, `life_date`, `source_date`, `captured_at`, `original_date`, `date`) over `created_at` when assigning a thought to a calendar year. This allows imported historical records with their own timestamps to be placed correctly in the timeline.
- **Thread eligibility gate** (`backfill-gmail-wikis.mjs`): A thread must meet `total_thread_word_count >= 500` AND (`distinct_messages >= 2` OR `atom_count >= 3`) to be synthesized. Word count is computed carefully to avoid double-counting when a thread contains both atomized and non-atomized messages — each `gmail_id` is credited only once unless per-atom word counts are present.
- **Resume-safe JSONL state log**: `backfill-gmail-wikis.mjs` appends one JSON record per processed thread to `./data/wiki-synthesis-state.jsonl`. Statuses are `ok`, `ok_partial_edges`, `failed`, and `skipped_ineligible`. Re-runs skip threads with `ok`/`ok_partial_edges` unless `--re-evaluate` is passed; threads with `>= 3` failed attempts are silently skipped.
- **`derived_from` edges**: After inserting a wiki thought, `backfill-gmail-wikis.mjs` writes `thought_edges` rows linking the wiki to every source atom with `relation: "derived_from"`. Edge inserts use `Promise.allSettled` so partial failure is tolerated and recorded; a wiki with zero successful edges is re-queued rather than accepted as `ok`.
- **Prompt injection defense**: Both scripts wrap untrusted thought/email content in explicit XML delimiters (`<entries>`, `<thread>`) and include system-prompt instructions directing the LLM to treat that block as data only, never as instructions. This is a deliberate mitigation against indirect prompt injection from captured third-party content.

## Non-Obvious Details

- `captureWikiThought` in `backfill-gmail-wikis.mjs` first deletes any existing `gmail_wiki` thoughts for the same `thread_id` before inserting a new one. This means re-synthesis replaces rather than duplicates wikis; cascading deletes on `thought_edges` clean up stale `derived_from` links automatically (assuming FK cascade is configured in the schema).
- The script prefers the `upsert_thought` RPC when available (shipped by the knowledge-graph recipe), but detects a 404 "function not found" response and falls back to a plain PostgREST insert. Any non-404 error from the RPC is re-thrown rather than silently swallowed.
- Sensitivity tier on the synthesized wiki thought is set to the most-restrictive tier among all source atoms (`restricted > personal > standard`), preventing a synthesis from downgrading the sensitivity of its source material.
- `synthesize-wiki.mjs` caps per-year prompt size at 300 entries to bound LLM context usage; `backfill-gmail-wikis.mjs` has no per-thread cap but the eligibility gate and `--limit` flag provide practical bounds.
- Both scripts load `.env.local` from the current working directory before falling back to process environment variables, allowing project-local credential files without shell export.

## Related Modules

- **[.claude/skills](../../../.claude/skills/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Prompt injection defense)
- **[.github](../../../.github/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue (plugin map), Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[.github/workflows](../../../.github/workflows/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue (plugin map), Two-stage review pipeline)
- **[extensions](../../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Resume-safe JSONL state log)
- **[extensions/family-calendar](../../../extensions/family-calendar/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, recurring vs. one-time activities in a unified table)
- **[extensions/home-maintenance](../../../extensions/home-maintenance/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Life-date bucketing, frequency_days=NULL for one-time tasks, trigger-driven next_due recalculation)
- **[integrations](../../../integrations/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Symmetric relation canonical ordering, derived_from edges)
- **[integrations/entity-extraction-worker](../../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (derived_from edges, support_count, symmetric relation canonicalization)
- **[integrations/entity-extraction-worker/_shared](../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Synthesizer catalogue (plugin map))
- **[recipes/chatgpt-conversation-import](../../chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Thread eligibility gate)
- **[recipes/email-history-import](../../email-history-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Noise filtering, Thread eligibility gate)
- **[recipes/entity-wiki](../../entity-wiki/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, derived_from edges)
- **[recipes/fingerprint-dedup-backfill](../../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Resume-safe JSONL state log)
- **[recipes/google-activity-import](../../google-activity-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Thread eligibility gate)
- **[recipes/grok-export-import](../../grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Thread eligibility gate, Transcript assembly)
- **[recipes/infographic-generator](../../infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Resume-safe JSONL state log)
- **[recipes/journals-blogger-import](../../journals-blogger-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Thread eligibility gate)
- **[recipes/life-engine](../../life-engine/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Dynamic loop rescheduling, Resume-safe JSONL state log)
- **[recipes/live-retrieval](../../live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility gate, Topic shift detection)
- **[recipes/ob-graph](../../ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (derived_from edges, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/obsidian-vault-import](../../obsidian-vault-import/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Secret scanning)
- **[recipes/panning-for-gold](../../panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Thread eligibility gate)
- **[recipes/perplexity-conversation-import](../../perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility gate, Two-sheet import (Conversations + Memory))
- **[recipes/research-to-decision-workflow](../../research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Skip rules, Thread eligibility gate)
- **[recipes/schema-aware-routing](../../schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Synthesizer catalogue (plugin map))
- **[recipes/thought-enrichment](../../thought-enrichment/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Synthesizer catalogue (plugin map))
- **[recipes/typed-edge-classifier](../../typed-edge-classifier/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), derived_from edges)
- **[recipes/wiki-compiler](../../wiki-compiler/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Resume-safe JSONL state log)
- **[recipes/wiki-synthesis](../CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility (content-weight gating), Thread eligibility gate)
- **[recipes/work-operating-model-activation](../../work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Thread eligibility gate)
- **[schemas](../../../schemas/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), derived_from edges)
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (derived_from edges, metadata-overlap thought connections)
- **[schemas/entity-extraction](../../../schemas/entity-extraction/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, derived_from edges)
- **[schemas/typed-reasoning-edges](../../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), derived_from edges, support_count evidence accumulation)
- **[skills](../../../skills/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Resume-safe JSONL state log)
- **[skills/heavy-file-ingestion](../../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Synthesizer catalogue (plugin map))
- **[skills/heavy-file-ingestion/scripts](../../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Resume-safe JSONL state log)
- **[skills/n-agentic-harnesses](../../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Synthesizer catalogue (plugin map))
- **[skills/panning-for-gold](../../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread eligibility gate)
- **[skills/weekly-signal-diff](../../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Thread eligibility gate)
- **[skills/world-model-diagnostic](../../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Simulated judgment)
