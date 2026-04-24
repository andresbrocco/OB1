# CONTEXT.md — Obsidian Vault Import

## Purpose

Imports an Obsidian markdown vault into Open Brain by walking `.md` files, parsing frontmatter and wikilinks, chunking notes into atomic thoughts, generating vector embeddings via OpenRouter, and inserting into the Supabase `thoughts` table. Designed for bulk one-time migration as well as incremental re-runs on changed notes.

## Responsibility Boundaries

- **Owns**: Vault traversal, note parsing, chunking strategy, secret scanning, embedding generation, Supabase insert, sync log management
- **Delegates to**: OpenRouter API (embeddings via `openai/text-embedding-3-small`, LLM distillation via `openai/gpt-4o-mini`), Supabase REST API for insert
- **Does not handle**: Obsidian graph or canvas files, attachment files, bidirectional link resolution, or post-import search/retrieval

## Key Concepts

- **Atomic thought**: The unit of import — a standalone, self-contained chunk of text prefixed with `[Obsidian: {title} | {folder} > {section}]` to preserve context without requiring the reader to know the source note
- **Two-tier chunking**: Notes under 500 words become one thought; longer notes are split on headings. Sections exceeding 1,000 words are further distilled by LLM into 1–3 sub-thoughts
- **Dual deduplication**: A local `obsidian-sync-log.json` file tracks content hashes per note path to skip unchanged notes on re-runs. A SHA-256 `content_fingerprint` on the thought text is sent to Supabase for DB-level dedup via `resolution=merge-duplicates`
- **Secret scanning**: Ten regex patterns screen each thought's content before embedding or inserting; matching thoughts are silently skipped and counted

## Non-Obvious Details

- The sync log (`obsidian-sync-log.json`) is written to the recipe directory, not the vault. It only records notes that produced at least one successful insert, so a note that failed mid-insert will be retried on the next run.
- Inline `#tags` are extracted from note bodies with false-positive suppression: code fences, inline code, HTML comments, and HTML tags are stripped before the tag regex runs.
- Frontmatter dates are tried under four key names (`date`, `created`, `created_at`, `date_created`); falling back to file `mtime`. The resulting date is used as the thought's `created_at` timestamp in Supabase.
- The `--after` filter operates on file `mtime`, not the frontmatter date — so notes edited after the cutoff but originally written before it will be included.
- Preflight checks run before any vault walking: Supabase table reachability and an OpenRouter embedding test call. This surfaces misconfigured credentials before a long import begins.
- Abort threshold: 10 consecutive Supabase insert failures trigger an early exit to avoid silent data loss during network or quota issues.
- The `Templates/` folder (case-insensitive) is skipped by heuristic, not by Obsidian configuration, so nested template folders with different names are not excluded automatically.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Secret scanning)
- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Idempotent PR comments)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Smart ingest auto-routing heuristic)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Auto-routing heuristic (shouldExtract))
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Dry-run two-phase ingestion, Ingestion modes (auto/single/extract))
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (AddToBrainMode, Atomic thought)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Two-tier chunking (heading split + LLM fallback))
- **[integrations](../../integrations/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Capture integration)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), re-extraction idempotency)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Structured capture format, prepareThoughtPayload)
- **[primitives](../../primitives/CONTEXT.md)** — Shares PR Review and CI Automation domain (Curation gate, Secret scanning)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Sync log)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Sync log, Two-layer dedup)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Secret scanning)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Dual deduplication (sync log + content fingerprint), Duplicate row)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Dual deduplication (sync log + content fingerprint))
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Dual deduplication (sync log + content fingerprint))
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Manifest file, Two-tier chunking (heading split + LLM fallback))
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Dual deduplication (sync log + content fingerprint))
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Dual deduplication (sync log + content fingerprint))
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Dual deduplication (sync log + content fingerprint))
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Session-scoped deduplication)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Local sync log deduplication)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Parallel capture pipeline)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, captureThought pipeline)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, captureThought pipeline)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Compile manifest, Compiled wiki, Two-tier chunking (heading split + LLM fallback))
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Two-tier chunking (heading split + LLM fallback), wiki vs entity-wiki distinction)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Secret scanning)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Dual deduplication (sync log + content fingerprint))
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), idempotent schema migration)
- **[server](../../server/CONTEXT.md)** — Shares Thought Ingestion and Capture domain (Atomic thought, Two-step capture (upsert + embedding patch), upsert_thought RPC (deduplication-aware insert))
- **[skills](../../skills/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Output Contract, Two-tier chunking (heading split + LLM fallback))
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Open Brain deduplication workflow)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Output directory convention (.ob1/), Two-tier chunking (heading split + LLM fallback))
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (.ob1 output directory, Two-tier chunking (heading split + LLM fallback))
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares PR Review and CI Automation domain (Approval gates, Secret scanning)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, Secret scanning)
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Prompt Injection and Security domain (Secret scanning, Simulated judgment)
