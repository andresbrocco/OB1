# CONTEXT.md — Heavy File Ingestion

## Purpose

Converts heavyweight files (PDF, DOCX, PPTX, XLSX, CSV, TSV) into cheap agent-friendly artifacts — markdown, per-sheet CSV, and a structured index — before any model tokens are spent on the original file. The core idea is that a deterministic conversion pass is almost always cheaper and faster than feeding raw files to a model, and the index it produces tells the agent whether escalation to a more expensive model is even justified.

## Responsibility Boundaries

- **Owns**: File-type detection, converter selection, artifact generation, quality flag emission, and index production (`index.md` + `index.json`)
- **Delegates to**: `pdfplumber`, `python-docx`, `python-pptx`, `openpyxl`, and optionally `markitdown` for the actual byte-level extraction
- **Does not handle**: Embedding the extracted content into Open Brain, downstream reasoning or summarization, or OCR of scanned images (flagged for escalation instead)

## Key Concepts

- **Deterministic-first policy**: A native converter (no model involved) always runs first. A model fallback is only justified when quality flags indicate the deterministic pass lost too much structure.
- **Cost tier escalation**: Three tiers — (1) deterministic converter + index, (2) cheap model on the extracted artifact only, (3) expensive model only after the file is already compressed. Escalation is driven by quality flags, not instinct.
- **Quality flags**: Strings such as `scanned_pdf_suspected`, `low_text_density`, `low_text_output`, `conversion_failed`, and `dependency_missing` emitted into `index.json`. The `recommended_next_step` field derives from these flags and tells the calling agent what to do next.
- **Converter preference**: The `--prefer` flag (`auto`, `native`, `markitdown`) controls whether `markitdown` is tried before the bundled native extractors. `auto` checks for `markitdown` on `$PATH` first; `native` skips it entirely.
- **Output directory convention**: Artifacts land in `<source_file>.ob1/` by default, keeping extracted content co-located with the source.
- **Variants**: Three client-specific skill packs (`variants/claude-code`, `variants/claude-desktop`, `variants/codex`) contain tailored `SKILL.md` prompts. `scripts/build_client_exports.py` assembles them into distributable ZIP/`.skill` archives under `resources/`.

## Non-Obvious Details

- The XLSX converter uses `read_only=True, data_only=True` in openpyxl, which means formula strings are not extracted — only computed cell values. Files that rely on uncomputed formulas may appear empty.
- PPTX conversion deduplicates text blocks per slide (to avoid repeating the slide title as body text), so intentional repeated text on a slide will be silently collapsed.
- DOCX table extraction is capped at 11 rows per table for token efficiency; a truncation notice is injected into the markdown when rows are omitted.
- The scanned-PDF heuristic fires when fewer than 70% of pages yield extractable text, or when average chars-per-page falls below 120 on a document of 3+ pages. These thresholds are hardcoded in `convert_pdf`.
- `build_client_exports.py` writes ZIP archives to `resources/` at the repo root. The `claude-desktop` variant is packaged as a `.skill` file (which is also a ZIP) rather than a `.zip` to match Claude Desktop's import format.
- The `uv run --with ...` invocation pattern in `SKILL.md` is the intended zero-setup entry point; it fetches dependencies ephemerally without polluting any existing environment.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, PR quota enforcement)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, Cost tier escalation)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), REST API connection pattern (Next.js dashboard), iron-session cookie auth)
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, SSR auth)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, iron-session cookie auth)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Quality flags)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, SvelteKit locals augmentation)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Auth guard via layout.server.ts, Client variants and build exports, latestResultKey highlight)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Output directory convention (.ob1/))
- **[integrations](../../integrations/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, Cost tier escalation)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, ExtractionCostCapError, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, Quality flags)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Portable Bundle)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Deterministic-first policy)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, Quality flags)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Body cleaning pipeline, Converter preference (auto/native/markitdown))
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output directory convention (.ob1/), Output modes (file / entity-metadata / thought))
- **[recipes/grok-export-import](../../recipes/grok-export-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), MongoDB-style date parsing)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Output directory convention (.ob1/))
- **[recipes/instagram-import](../../recipes/instagram-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Meta latin1/UTF-8 encoding repair)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Converter preference (auto/native/markitdown))
- **[recipes/local-ollama-embeddings](../../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Output directory convention (.ob1/), Two-tier chunking (heading split + LLM fallback))
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Selective LLM summarization)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Schema-aware routing)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, LLM classification prompt with importance/confidence calibration)
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, Hard cost cap with proactive parallelism clamping)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, In-memory rate limiter with cold-start reset)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, in-memory sliding-window rate limiter)
- **[recipes/wiki-compiler](../../recipes/wiki-compiler/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Output directory convention (.ob1/))
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Synthesizer catalogue)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Synthesizer catalogue (plugin map))
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[recipes/x-twitter-import](../../recipes/x-twitter-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Tweet batching, Twitter JS export format)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality flags, importance/quality_score ranking signals)
- **[skills](../CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, Variants)
- **[skills/claudeception](../claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, Quality flags)
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, Quality flags)
- **[skills/heavy-file-ingestion/scripts](scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Converter preference (auto/native/markitdown), export bundles)
- **[skills/n-agentic-harnesses](../n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Mode classification)
- **[skills/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output directory convention (.ob1/), Permanent file write discipline)
- **[skills/work-operating-model](../work-operating-model/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Export artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[skills/world-model-diagnostic](../world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Quality flags)
