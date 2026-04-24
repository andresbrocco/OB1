# CONTEXT.md — Infographic Generator

## Purpose

Converts research documents, Open Brain thought content, or raw analysis into professional infographic images via a two-phase pipeline: an AI client (Claude) writes structured image prompts, then a Python script submits those prompts to the Google Gemini image generation API and saves the resulting PNG files locally.

## Responsibility Boundaries

- **Owns**: Prompt structure specification, content chunking rules, audience calibration logic, image generation script execution, and output manifest creation
- **Delegates to**: The AI client (Claude) for prompt authoring and content analysis; the Gemini API for actual image rendering
- **Does not handle**: Hosting or serving generated images; embedding images into documents; any database writes (Open Brain storage is optional and user-initiated)

## Key Concepts

- **Two-phase pipeline**: The skill file (`infographic-generator.skill.md`) governs the AI client's behavior through Steps 1–7; `generate.py` is the execution layer called at Step 6. The skill writes a structured markdown prompts file first, then the script reads and executes it — these are decoupled so prompts can be reviewed or edited before generation.
- **Prompts file format**: The intermediate artifact is a markdown file under `./infographic-prompts/` with `## Infographic N: Title` headers and `### Prompt` subsections. `generate.py` parses this format via regex; any deviation breaks parsing silently (section is skipped).
- **Manifest file**: After generation `generate.py` writes `media/_latest_generation.json` listing all successfully generated images. The skill reads this manifest to display results to the user.
- **`--redo N` flag**: Allows regenerating a single infographic by 1-based index without re-running the full pipeline, preserving all other previously generated images.

## Non-Obvious Details

- `generate.py` self-bootstraps by adding the recipe-local `.venv/lib/python*/site-packages` to `sys.path` before importing `google.genai`. If no `.venv` exists, it falls back to the system Python environment — no error is raised on missing venv, only on missing `google-genai` package.
- The model constants (`MODEL_FREE = "gemini-2.5-flash-image"`, `MODEL_PREMIUM = "gemini-3.1-flash-image-preview"`) are hardcoded strings. If Gemini renames these endpoints the script will fail at API call time, not at startup.
- The skill file specifies the script path as `~/.claude/skills/infographic-generator/generate.py` — this assumes the recipe has been installed as a Claude skill, not run directly from the repo clone location.
- Aspect ratio defaults to `"4:5"` (portrait/phone) when not specified in a prompt block; layout-type decisions (landscape vs. portrait) must be made explicitly in each prompt to override this.
- The `--redo` progress counter in `generate.py` uses `args.redo` for the display number but `i + 1` for the progress fraction — when redoing a single item, progress always shows `[1/1]` regardless of the original index.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompts file format, Skill file format)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Two-phase pipeline)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Manifest file)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Manifest file)
- **[recipes](../CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompts file format, Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Two-phase pipeline)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Manifest file, Pyramid summaries)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Retrospective Mode, Two-phase pipeline)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Output modes (file / entity-metadata / thought))
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (--redo flag, Cursor-based resumability)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Self-improvement protocol, Two-phase pipeline)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Manifest file, Two-tier chunking (heading split + LLM fallback))
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Prompts file format)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (--redo flag, Cursor-based resumable state)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Two-phase pipeline)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Phase toggles, Two-phase pipeline)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Resume-safe JSONL state log)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (--redo flag, Checkpoint + Entry Separation)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Two-phase pipeline)
- **[skills](../../skills/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Output Contract)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Retrospective mode, Two-phase pipeline)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Output directory convention (.ob1/))
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Manifest file)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Two-phase pipeline)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Permanent file write discipline)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Prompts file format)
