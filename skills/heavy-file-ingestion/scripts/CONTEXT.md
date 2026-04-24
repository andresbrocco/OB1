# CONTEXT.md — Scripts

## Purpose

Provides two executable Python scripts: one that converts heavyweight files (PDF, DOCX, PPTX, XLSX, CSV, TSV, plain text) into AI-consumable markdown and CSV artifacts with a structured index, and one that assembles distributable client export bundles (`.zip` and `.skill` archives) for each supported AI client variant.

## Responsibility Boundaries

- **Owns**: File format detection, per-format conversion logic, quality flag emission, index file generation (`index.json` / `index.md`), and export bundle assembly
- **Delegates to**: Optional external tools (`markitdown` CLI via subprocess, `uvx`) and optional Python libraries (`openpyxl`, `python-pptx`, `python-docx`, `pdfplumber`) loaded lazily at runtime
- **Does not handle**: Uploading artifacts to any storage system, AI model interaction, or scheduling

## Key Concepts

- **ConversionResult**: Central dataclass that accumulates artifacts, warnings, quality flags, and stats for a single conversion run. All converter functions return one.
- **Quality flags**: String tokens appended to a result to signal degraded output — e.g., `scanned_pdf_suspected`, `low_text_density`, `low_text_output`, `conversion_failed`, `dependency_missing`. The `infer_next_step` function maps these flags to a recommended action string consumed by the calling AI agent.
- **`prefer` strategy**: CLI flag (`auto` / `native` / `markitdown`) that controls whether the script attempts the `markitdown` external converter before falling back to native Python libraries.
- **`.ob1` output directory**: By default, converted artifacts are written to `<source>.ob1/` alongside the source file.
- **Export bundles**: `build_client_exports.py` assembles variant-specific directories (from `variants/`) plus the shared `convert_heavy_file.py` script and `references/` folder into `.zip` (Claude Code, Codex) or `.skill` (Claude Desktop) archives, placing them in the repo-level `resources/` directory.

## Non-Obvious Details

- `convert_heavy_file.py` exits with code `2` (not `1`) when conversion fails, so callers can distinguish a conversion failure from a usage error.
- The `markitdown` path is attempted first for DOCX and PDF only; XLSX and PPTX always use native libraries regardless of the `prefer` flag.
- Table rows in DOCX are truncated at 10 data rows per table to limit token consumption.
- `build_client_exports.py` writes intermediate files into a `.build-exports/` scratch directory inside the skill root, then zips from there; the scratch directory is reset on every run.
- The `.skill` file for Claude Desktop is a ZIP archive with a `.skill` extension — it is not a separate binary format.

## Related Modules

- **[dashboards/open-brain-dashboard-next](../../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (quality audit (quality_score <= 29), quality flags)
- **[dashboards/open-brain-dashboard-next/app/api](../../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, quality flags)
- **[dashboards/open-brain-dashboard/src/routes](../../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (.ob1 output directory, Incremental result merging)
- **[extensions](../../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, AGENT_SPEC.md machine-readable generation spec)
- **[integrations/entity-extraction-worker/_shared](../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), quality flags)
- **[recipes/adaptive-capture-classification](../../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, quality flags)
- **[recipes/bring-your-own-context](../../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Portable Bundle, export bundles)
- **[recipes/chatgpt-conversation-import](../../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (.ob1 output directory, Pyramid summaries)
- **[recipes/claudeception](../../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, quality flags)
- **[recipes/email-history-import](../../../recipes/email-history-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Body cleaning pipeline, ConversionResult, export bundles)
- **[recipes/entity-wiki](../../../recipes/entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Output modes (file / entity-metadata / thought))
- **[recipes/grok-export-import](../../../recipes/grok-export-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, MongoDB-style date parsing, export bundles)
- **[recipes/infographic-generator](../../../recipes/infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Manifest file)
- **[recipes/instagram-import](../../../recipes/instagram-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Meta latin1/UTF-8 encoding repair, export bundles)
- **[recipes/journals-blogger-import](../../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, ConversionResult, export bundles)
- **[recipes/local-ollama-embeddings](../../../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Multi-format input (stdin, positional args, .txt, .jsonl), export bundles)
- **[recipes/obsidian-vault-import](../../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (.ob1 output directory, Two-tier chunking (heading split + LLM fallback))
- **[recipes/thought-enrichment](../../../recipes/thought-enrichment/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (LLM classification prompt with importance/confidence calibration, quality flags)
- **[recipes/wiki-compiler](../../../recipes/wiki-compiler/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Compile manifest)
- **[recipes/wiki-synthesis](../../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Resume-safe JSONL state log)
- **[recipes/work-operating-model-activation](../../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), export bundles)
- **[recipes/x-twitter-import](../../../recipes/x-twitter-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Tweet batching, Twitter JS export format, export bundles)
- **[schemas/enhanced-thoughts](../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (importance/quality_score ranking signals, quality flags)
- **[skills](../../CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Output Contract)
- **[skills/claudeception](../../claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, quality flags)
- **[skills/financial-model-review](../../financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, quality flags)
- **[skills/heavy-file-ingestion](../CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Converter preference (auto/native/markitdown), export bundles)
- **[skills/panning-for-gold](../../panning-for-gold/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Permanent file write discipline)
- **[skills/work-operating-model](../../work-operating-model/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), export bundles)
- **[skills/world-model-diagnostic](../../world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, quality flags)
