# CONTEXT.md — Claudeception

## Purpose

A meta-learning recipe that enables an AI agent to extract reusable knowledge from work sessions and codify it into new skill files. Rather than solving a domain problem directly, it teaches the agent how to recognize, structure, and persist its own discoveries — creating a self-improving knowledge loop integrated with Open Brain.

## Responsibility Boundaries

- **Owns**: The protocol for evaluating, structuring, saving, and capturing newly discovered skills
- **Delegates to**: Open Brain (`search_thoughts`, `capture_thought`) for deduplication and cross-session retrieval; the filesystem for skill file storage
- **Does not handle**: Executing the skills it creates; managing the Open Brain schema; UI or scheduling of extraction

## Key Concepts

- **Aiception / Claudeception**: The meta-pattern of an AI skill that generates other skills — "skills that create skills." The recipe was renamed from Claudeception to Aiception but the folder retains the original name.
- **Extraction**: The deliberate process of identifying non-obvious, reusable knowledge from a completed task and writing it into a structured skill file.
- **Retrospective Mode**: An on-demand invocation (`/aiception`) that reviews an entire session for extraction candidates rather than triggering inline.
- **Quality Gate**: A seven-point checklist that guards against over-extraction, vague descriptions, unverified solutions, and duplication before a skill is committed.
- **Skill Lifecycle**: Creation → Refinement → Deprecation → Archival. Skills are expected to evolve or be removed as tools change.

## Non-Obvious Details

- The skill's YAML frontmatter uses the name `aiception` (not `claudeception`), and its description is intentionally kept to a single line with no pipe (`|`) character — multi-line descriptions silently break agent routing.
- Open Brain must be searched *before* creating a new skill and captured *after*, in that order. Skipping either step defeats the deduplication and cross-session discovery that justify this recipe's existence.
- The recipe defines two distinct save locations (`.claude/skills/` for project scope, `~/.claude/skills/` for user-wide), and the choice between them is a deliberate decision the agent must make per extraction.
- Automatic triggers fire after tasks meeting time/effort thresholds (e.g., >10 minutes of undocumented investigation) — the agent is expected to invoke this without an explicit user command in those cases.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Skill Lifecycle, Skill file format)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Artifact handoff between workflows, Retrospective Mode)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, quality audit (quality_score <= 29))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Quality Gate)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Retrospective Mode)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Post-search filter extraction)
- **[docs](../../docs/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Progressive onboarding sequence, Retrospective Mode)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Skill Lifecycle)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, ExtractionCostCapError, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, _enrichment_status)
- **[recipes](../CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Recipe vs. Extension distinction, Recipe vs. Skill distinction, Skill Lifecycle, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Retrospective Mode, Two-phase pipeline)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Two-Prompt Extraction Sequence)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Output modes (file / entity-metadata / thought), Skill Lifecycle)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Retrospective Mode, Two-phase pipeline)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Retrospective Mode, Self-improvement protocol)
- **[recipes/repo-learning-coach](../repo-learning-coach/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, RepoLearningConfig, Retrospective Mode)
- **[recipes/repo-learning-coach/server](../repo-learning-coach/server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Retrospective Mode)
- **[recipes/repo-learning-coach/src](../repo-learning-coach/src/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Retrospective Mode)
- **[recipes/repo-learning-coach/src/lib](../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Retrospective Mode)
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Skill Lifecycle)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Pending person confirmation, Three-pass person resolution)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Extraction)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Parallel capture pipeline, Retrospective Mode)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Phase toggles, Retrospective Mode)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, source_confidence)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Extraction)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, importance/quality_score ranking signals)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Async queue with content-addressed re-queue, Retrospective Mode)
- **[skills](../../skills/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Retrospective Mode)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Aiception/Claudeception (self-referential skill extraction), Retrospective Mode, Retrospective mode)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, Quality Gate)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, Quality flags)
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, quality flags)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Approval gates, Harness, Harness primitives, Retrospective Mode)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Thread extraction)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Quality Gate)
