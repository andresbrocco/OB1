# CONTEXT.md — Claudeception (Aiception)

## Purpose

A meta-skill that teaches an AI agent to extract reusable knowledge from its own work sessions and codify that knowledge into new skills. It enables continuous, autonomous improvement by turning non-obvious discoveries — debugging breakthroughs, undocumented tool behaviors, project-specific patterns — into structured skill files that future sessions can load.

## Responsibility Boundaries

- **Owns**: The decision logic for when knowledge is worth extracting, the structured format for new skill files, and the workflow for capturing extracted skills into Open Brain
- **Delegates to**: Open Brain (`search_thoughts`, `capture_thought`) for duplicate detection and cross-session persistence; the AI client's skills system (`.claude/skills/`) for skill file storage
- **Does not handle**: Skill retrieval or invocation at task time — that is the responsibility of the AI client loading the skill files

## Key Concepts

- **Aiception / Claudeception**: The self-referential nature of the skill — an AI skill about creating AI skills. The directory is named `claudeception` but the skill itself was renamed to `aiception` at v2.0.0 for client-agnosticism.
- **Extraction threshold**: Not every task produces a skill. The skill defines explicit quality gates (reusable, non-trivial, specific, verified) and automatic trigger conditions (e.g., >10 minutes of investigation, misleading error messages).
- **Deduplication against Open Brain**: Before creating a new skill, the agent must search Open Brain for existing knowledge. The three-way decision (update existing / create with cross-reference / create new) prevents knowledge fragmentation.
- **Skill lifecycle**: Creation → Refinement → Deprecation → Archival. Deprecation is explicit to prevent stale skills from surfacing.
- **Retrospective mode**: `/aiception` invoked at session end triggers a structured review of the full session for extraction candidates, distinct from inline extraction during active work.

## Non-Obvious Details

- The skill file frontmatter uses `name: aiception` while the directory remains `skills/claudeception` — a legacy naming inconsistency from the v2.0.0 rename.
- Skill storage has two scopes: project-level (`.claude/skills/`) and user-wide (`~/.claude/skills/`). The skill instructs the agent to choose based on specificity, but this distinction depends on the AI client supporting both paths.
- After creating a skill file, the agent must also call `capture_thought` to record its existence in Open Brain. This two-step save (file + brain) is required for cross-session discoverability; omitting the brain capture means future sessions won't find the skill via semantic search.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Skill file format, Skill lifecycle (creation to archival))
- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Open Brain deduplication workflow)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Artifact handoff between workflows, Retrospective mode)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, quality audit (quality_score <= 29))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Extraction threshold and quality gates)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Retrospective mode)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Post-search filter extraction)
- **[docs](../../docs/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Progressive onboarding sequence, Retrospective mode)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Dual-scope skill storage, Learning path (learning_order 1-6), Skill lifecycle (creation to archival))
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, re-extraction idempotency)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, _enrichment_status)
- **[recipes](../../recipes/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Recipe vs. Extension distinction, Recipe vs. Skill distinction, Skill lifecycle (creation to archival), _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Retrospective mode, Two-phase pipeline)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Two-Prompt Extraction Sequence)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, Sync log)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Aiception/Claudeception (self-referential skill extraction), Retrospective Mode, Retrospective mode)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, Sync log, Two-layer dedup)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Output modes (file / entity-metadata / thought), Skill lifecycle (creation to archival))
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Open Brain deduplication workflow)
- **[recipes/google-activity-import](../../recipes/google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Open Brain deduplication workflow)
- **[recipes/grok-export-import](../../recipes/grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Open Brain deduplication workflow)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Retrospective mode, Two-phase pipeline)
- **[recipes/instagram-import](../../recipes/instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Open Brain deduplication workflow)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Open Brain deduplication workflow)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Retrospective mode, Self-improvement protocol)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, Session-scoped deduplication)
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Open Brain deduplication workflow)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Local sync log deduplication, Open Brain deduplication workflow)
- **[recipes/repo-learning-coach](../../recipes/repo-learning-coach/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, RepoLearningConfig, Retrospective mode)
- **[recipes/repo-learning-coach/server](../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Retrospective mode)
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Retrospective mode)
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Retrospective mode)
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Prompt stubs, Skill lifecycle (creation to archival))
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Pending person confirmation, Three-pass person resolution)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Extraction threshold and quality gates)
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Open Brain deduplication workflow)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Parallel capture pipeline, Retrospective mode)
- **[recipes/wiki-compiler](../../recipes/wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Phase toggles, Retrospective mode)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, source_confidence)
- **[recipes/x-twitter-import](../../recipes/x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Open Brain deduplication workflow)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Extraction threshold and quality gates)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, idempotent schema migration)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Async queue with content-addressed re-queue, Retrospective mode)
- **[skills](../CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Retrospective mode)
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, Fatal issues vs. caution flags vs. acceptable simplifications)
- **[skills/heavy-file-ingestion](../heavy-file-ingestion/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, Quality flags)
- **[skills/heavy-file-ingestion/scripts](../heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, quality flags)
- **[skills/n-agentic-harnesses](../n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Approval gates, Harness, Harness primitives, Retrospective mode)
- **[skills/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Thread extraction)
- **[skills/work-operating-model](../work-operating-model/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, source_confidence (confirmed vs synthesized))
- **[skills/world-model-diagnostic](../world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, Five-principle evaluation)
