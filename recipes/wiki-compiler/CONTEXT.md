# CONTEXT.md — Wiki Compiler

## Purpose

Orchestrates a multi-phase pipeline that transforms raw Open Brain thought data into a structured compiled wiki. It sequences four distinct recipe scripts — entity extraction, typed-edge classification, entity wiki generation, and topic wiki synthesis — into a single runnable command with coordinated output, phase toggles, and a manifest artifact.

## Responsibility Boundaries

- **Owns**: Pipeline sequencing, phase skip/require logic, argument parsing, manifest creation, and output directory layout
- **Delegates to**: `typed-edge-classifier/classify-edges.mjs`, `entity-wiki/generate-wiki.mjs`, `wiki-synthesis/synthesize-wiki.mjs`, `wiki-synthesis/backfill-gmail-wikis.mjs` (each recipe handles its own domain logic)
- **Does not handle**: LLM calls, database reads/writes, embedding generation, or any content synthesis — all of that lives in the delegated scripts

## Key Concepts

- **Compiled wiki**: The end-to-end output of all four phases written to a local `compiled-wiki/` directory (or `--out-dir`), as opposed to any single recipe's incremental output.
- **Phase toggles**: Each pipeline step can be individually skipped (`--skip-extraction`, `--skip-edges`, `--skip-entity-wiki`, `--skip-topic-wiki`), allowing partial re-runs without repeating expensive steps.
- **Best-effort mode**: `--best-effort` allows the pipeline to continue past a failing phase and record the failure in the manifest instead of aborting.
- **Compile manifest**: `compile-manifest.json` written to `--out-dir` after every run; records each step's status, timestamp, and key parameters — the authoritative record of what a given run did.
- **Topic**: A named synthesis strategy passed to `wiki-synthesis`. Defaults to `"autobiography"` if no `--topic` flag is provided. Multiple `--topic` flags run each topic in sequence.

## Non-Obvious Details

- Entity extraction is the only phase that makes a remote HTTP call (to `ENTITY_EXTRACTION_WORKER_URL` or the Supabase edge function derived from `OPEN_BRAIN_URL`). All other phases spawn local Node.js child processes.
- If `ENTITY_EXTRACTION_WORKER_URL` or the access key is missing, extraction is silently skipped by default; pass `--require-extraction` to convert that into a hard failure.
- The script resolves all sibling recipe paths relative to `REPO_ROOT` at startup and will throw immediately if any dependency script is missing — before any phase runs.
- Environment variables are merged from `.env`, `.env.local`, and `process.env` in that order, with `process.env` winning. This lets `.env.local` override project defaults without affecting CI.
- `--scope key=value` flags are forwarded only to topic synthesis runs, not to edge classification or entity wiki phases.

## Related Modules

- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Phase toggles)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Compile manifest, Compiled wiki, Incremental result merging)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Compile manifest)
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Phase toggles, Two-phase pipeline)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Compile manifest, Compiled wiki, Pyramid summaries)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Phase toggles, Retrospective Mode)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Output modes (file / entity-metadata / thought))
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumability, Phase toggles)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Phase toggles, Two-phase pipeline)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Phase toggles, Self-improvement protocol)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Compile manifest, Compiled wiki, Two-tier chunking (heading split + LLM fallback))
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Cursor-based resumable state, Phase toggles)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Depends on for Two-stage Haiku/Opus hybrid classifier that populates thought_edges with typed semantic reasoning relations between thought pairs
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Phase toggles)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Depends on for LLM-driven wiki article synthesis from Open Brain atomic thoughts, with an autobiographical narrative synthesizer and a Gmail-thread summarizer
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Resume-safe JSONL state log)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Checkpoint + Entry Separation, Phase toggles)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Phase toggles)
- **[skills](../../skills/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Output Contract)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Phase toggles, Retrospective mode)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Output directory convention (.ob1/))
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Compile manifest)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Phase toggles)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Permanent file write discipline)
