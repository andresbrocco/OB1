# CONTEXT.md — Bring Your Own Context

## Purpose

Provides the context ingestion half of the portable-context workflow. Its role is to seed Open Brain with durable, well-structured raw context from two sources: existing AI memory (e.g., Claude, ChatGPT) and external note exports (e.g., Notion, Obsidian, Apple Notes). It is intentionally scoped to extraction only; the downstream normalization into final profile artifacts is handled by the `work-operating-model-activation` recipe.

## Responsibility Boundaries

- **Owns**: Extraction prompts for AI memory harvesting and external content import; the JSON schema defining the portable context bundle format (`context-profile.schema.json`).
- **Delegates to**: `work-operating-model-activation` for the normalization interview that produces `USER.md`, `SOUL.md`, `HEARTBEAT.md`, and `operating-model.json` from the raw seeded context.
- **Does not handle**: Profile synthesis, schedule generation, or any final artifact output — those are explicitly deferred to the second stage.

## Key Concepts

- **Operating Model Layers**: The structured taxonomy used in `operating-model.json` — `operating_rhythms`, `recurring_decisions`, `dependencies`, `institutional_knowledge`, and `friction`. Each layer has its own entry schema with fields like `source_confidence`, `cadence`, `trigger`, and `status`.
- **Source Confidence**: A two-value enum (`confirmed` vs. `synthesized`) on each entry, distinguishing verified user-stated facts from AI-inferred context.
- **Portable Bundle**: The complete export payload containing `operating-model.json`, `USER.md`, `SOUL.md`, `HEARTBEAT.md`, and `schedule-recommendations.json` — intended to be transferable across AI clients.
- **Two-Prompt Sequence**: Prompt 1 (Memory Extraction) pulls context from the AI's own memory; Prompt 2 (Context Import) ingests pasted or uploaded external material. Both must run before the Work Operating Model normalization pass begins.

## Non-Obvious Details

- The extraction prompts explicitly prohibit generating final profile artifacts (`USER.md`, `SOUL.md`, etc.). This boundary is enforced by prompt guardrails, not by code — contributors should not merge changes that blur the extraction/normalization boundary.
- `context-profile.schema.json` documents the `generate_operating_model_exports` tool's return contract. The `exports` object carries artifact contents as raw strings, not parsed objects; parsed contracts are defined separately under `$defs/portableBundleParsed`.
- If both extraction prompts apply, Prompt 1 must run before Prompt 2, as memory extraction establishes the base layer that import content then extends.
- This recipe has no runnable code of its own; it is composed entirely of prompts and a schema, and depends on the `work-operating-model` skill and the `deploy-edge-function` and `remote-mcp` primitives for full execution.

## Related Modules

- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Source Confidence, quality audit (quality_score <= 29))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Source Confidence)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, Two-Prompt Extraction Sequence)
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Interview stages, Operating Model Layers)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Two-Prompt Extraction Sequence, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Two-Prompt Extraction Sequence, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Two-Prompt Extraction Sequence, _enrichment_status)
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, Source Confidence)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Two-Prompt Extraction Sequence)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Body cleaning pipeline, Portable Bundle)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (MongoDB-style date parsing, Portable Bundle)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Meta latin1/UTF-8 encoding repair, Portable Bundle)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Portable Bundle)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Two-Prompt Extraction Sequence)
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (Multi-format input (stdin, positional args, .txt, .jsonl), Portable Bundle)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Operating Model Layers)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Three-pass person resolution, Two-Prompt Extraction Sequence)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Two-Prompt Extraction Sequence)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Depends on for Remote MCP server that elicits a user's work operating model across five structured layers and renders versioned, agent-ready export artifacts
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Portable Bundle, Tweet batching, Twitter JS export format)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Two-Prompt Extraction Sequence)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Source Confidence, importance/quality_score ranking signals)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, Two-Prompt Extraction Sequence)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Two-Prompt Extraction Sequence)
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Conviction state, Decision-readiness, Operating Model Layers)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Operating Model Layers, Structural risk vs. business risk)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Portable Bundle)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Portable Bundle, export bundles)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Thread extraction, Two-Prompt Extraction Sequence)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Operating Model Layers, Structural questions framework)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Depends on for Structured AI skill for interviewing users about how their work actually runs, saving approved results into Open Brain via paired recipe MCP tools, and generating agent-ready operating model exports
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Source Confidence)
