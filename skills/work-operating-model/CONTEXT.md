# CONTEXT.md — Work Operating Model

## Purpose

A structured AI skill that interviews a user through five fixed layers to surface and encode how their work actually runs, then saves the approved model into Open Brain and generates agent-ready export artifacts. The goal is explicit externalization of tacit work patterns before any automation is attempted.

## Responsibility Boundaries

- **Owns**: The interview protocol, layer sequencing, checkpoint/confirmation discipline, canonical entry schema enforcement, contradiction detection, and memory capture discipline
- **Delegates to**: The paired Work Operating Model recipe for persistence (`start_operating_model_session`, `save_operating_model_layer`, `query_operating_model`, `generate_operating_model_exports`) and to the base Open Brain tools (`search_thoughts`, `capture_thought`) for memory reads and lean thought capture
- **Does not handle**: Database schema, MCP server deployment, tool name resolution (the skill requires the AI client to discover tool names at runtime)

## Key Concepts

- **Five fixed layers**: The interview always proceeds in order — operating rhythms, recurring decisions, dependencies, institutional knowledge, friction. Order is non-negotiable.
- **source_confidence**: Each saved entry is tagged `confirmed` (user explicitly stated or approved verbatim) or `synthesized` (AI abstracted a pattern from multiple examples and user approved the abstraction). These are not interchangeable.
- **Canonical entry contract**: Every saved layer entry must include a fixed set of fields (`title`, `summary`, `cadence`, `trigger`, `inputs`, `stakeholders`, `constraints`, `details`, `source_confidence`, `status`, `last_validated_at`). Each layer type also has its own `details` sub-shape.
- **Export artifacts**: Final output is five files — `operating-model.json`, `USER.md`, `SOUL.md`, `HEARTBEAT.md`, `schedule-recommendations.json` — intended to be consumed by AI agents as context about how the user operates.

## Non-Obvious Details

- Retrieved memory hints from `search_thoughts` are explicitly treated as tentative prompts, not facts. Nothing retrieved from memory may be persisted unless the user independently confirms or approves a synthesized abstraction of it.
- The contradiction pass (Phase 3) is mandatory before export generation and may trigger a layer resave. Cross-layer tensions — such as claimed rhythms conflicting with reported friction — must be surfaced explicitly, not smoothed over.
- Memory capture is intentionally sparse: one summary thought per approved layer plus one final synthesis thought. Capturing per-entry thoughts is explicitly prohibited to avoid polluting the brain with redundant entries.
- The skill must stop and surface a clear error if any of its required MCP tools are absent. It does not degrade gracefully or skip tools silently.
- Interview prompts are anchored to recent concrete events ("last two weeks", "a real Monday") rather than abstract self-description, to counter the gap between how users think they work and how they actually work.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Skill file format)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (quality audit (quality_score <= 29), source_confidence (confirmed vs synthesized))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, source_confidence (confirmed vs synthesized))
- **[extensions](../../extensions/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Canonical entry contract, Learning path (learning_order 1-6))
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five fixed interview layers, Interview stages)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), source_confidence (confirmed vs synthesized))
- **[recipes](../../recipes/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, source_confidence (confirmed vs synthesized))
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Provides SKILL.md (prompt pack loaded by AI client) consumed by this module
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, source_confidence (confirmed vs synthesized))
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Body cleaning pipeline, Export artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Output modes (file / entity-metadata / thought))
- **[recipes/grok-export-import](../../recipes/grok-export-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), MongoDB-style date parsing)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Prompts file format)
- **[recipes/instagram-import](../../recipes/instagram-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), Meta latin1/UTF-8 encoding repair)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Export artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Flywheel (read side), Lean memory discipline)
- **[recipes/local-ollama-embeddings](../../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Five fixed interview layers)
- **[recipes/repo-learning-coach](../../recipes/repo-learning-coach/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Lean memory discipline, Understanding State)
- **[recipes/repo-learning-coach/server](../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Lean memory discipline, UnderstandingState)
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Lean memory discipline, UnderstandingState)
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridgeState, Lean memory discipline, UnderstandingState)
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Investor path)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (LLM classification prompt with importance/confidence calibration, source_confidence (confirmed vs synthesized))
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Provides SKILL.md (prompt pack loaded by AI client) consumed by this module
- **[recipes/x-twitter-import](../../recipes/x-twitter-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), Tweet batching, Twitter JS export format)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (importance/quality_score ranking signals, source_confidence (confirmed vs synthesized))
- **[skills](../CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, SKILL.md)
- **[skills/claudeception](../claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, source_confidence (confirmed vs synthesized))
- **[skills/deal-memo-drafting](../deal-memo-drafting/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Deal memo, Diligence packet, IC memo)
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Fatal issues vs. caution flags vs. acceptable simplifications, Investor-grade / Board-grade / Internal planning standard, Structural risk vs. business risk)
- **[skills/heavy-file-ingestion](../heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Export artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[skills/heavy-file-ingestion/scripts](../heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Export artifacts (USER.md, SOUL.md, HEARTBEAT.md), export bundles)
- **[skills/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Five fixed interview layers, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../weekly-signal-diff/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five fixed interview layers, Structural questions framework)
- **[skills/world-model-diagnostic](../world-model-diagnostic/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Firm finding / Inference / Open question labeling, Five-principle evaluation)
