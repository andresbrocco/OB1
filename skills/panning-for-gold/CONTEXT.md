# CONTEXT.md — Panning for Gold

## Purpose

A reusable AI client skill that transforms raw, unstructured captures (voice transcripts, brain dumps, stream-of-consciousness notes) into evaluated, prioritized idea inventories stored in Open Brain. The skill instructs an AI agent through a structured multi-phase process: save inputs to disk, clean speaker attribution, extract all idea threads exhaustively, evaluate high-signal ones, synthesize a gold-found file, and capture results to Open Brain.

## Responsibility Boundaries

- **Owns**: The extraction-to-evaluation workflow for unstructured multi-topic captures; agent behavior rules, triage logic, and output file conventions for panning sessions
- **Delegates to**: Open Brain (`capture_thought`, `search_thoughts`) for persistent storage; background evaluator agents for parallel idea evaluation; the AI client's file tools for writing intermediate and final outputs
- **Does not handle**: General session summaries, single-topic notes, or structured data imports — those belong to other skills or recipes

## Key Concepts

- **Panning / Gold-Found**: The metaphor for the workflow. Phase 1 "pans" (extracts all threads without filtering), Phase 2 evaluates for signal, Phase 3 synthesizes the "gold-found" file with verdicts.
- **Thread**: A discrete idea, observation, or topic extracted from the raw source. A 1-hour conversation typically yields 40–80+ threads; collapsing or skipping threads is the primary extraction failure mode.
- **Verdict taxonomy**: Each evaluated thread receives one of four dispositions — **ACT NOW**, **RESEARCH MORE**, **PARK IT**, or **KILL IT** — which drive the gold-found file structure.
- **Speaker Consolidation (Phase 0.5)**: A pre-extraction step specific to multi-speaker voice transcripts. Auto-generated speaker labels from transcription tools (Otter, Plaud, etc.) are unreliable and often actively misleading — a 2-person meeting can produce 10 label IDs. The skill resolves this via anchor-line identification and scene-based re-attribution before any thread extraction.
- **Mary's Law Check**: A synthesis checklist item asking whether there is a person to contact before writing more code — a heuristic for catching relationship-driven actions that purely technical triage misses.
- **COS items**: Chief-of-Staff-style action items (WAITING_FOR, Calendar, CRM Updates, Decisions) appended to the gold-found file.

## Non-Obvious Details

- **Permanent file writes are mandatory at every phase.** The Critical Rules section exists because agent memory and background task outputs do not survive context compaction. Phase 0 saves the raw input, Phase 1 saves the inventory, Phase 2 evaluators each write their own file, and Phase 3 synthesis is always written inline (never delegated to an agent) from files already on disk.
- **Summaries-first reading strategy.** If a meeting summary exists alongside a transcript, the skill reads the summary first and uses Grep for targeted quote lookups rather than re-reading the full transcript. This was added after a single session burned ~30K tokens re-reading a 926-line transcript unnecessarily.
- **Over-extraction is correct in Phase 1.** The default posture is completeness, not curation. Filtering by "actionable" or "technical" in Phase 1 is explicitly flagged as the number-one failure mode. Phase 2 triage handles prioritization.
- **Background evaluator cap.** No more than 5 background agents should be dispatched. More than 5 indicates a Phase 1 mis-triage. Inline evaluation is preferred for 1–3 threads.
- **Lessons Log in SKILL.md.** The skill is designed to self-improve: after each session the agent is instructed to update the lessons log and critical rules directly in the SKILL.md file. The log currently documents six production-derived lessons from March 2026.
- **Trigger phrases.** The YAML front matter in SKILL.md lists the phrases that should activate this skill: "pan for gold", "brain dump", "process this", "what did I say", and multi-topic markdown files.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Mary's Law Check)
- **[.github](../../.github/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, security-blocked and needs-maintainer-triage label lifecycle)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, pull_request vs pull_request_target security split)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban workflow (task/idea types only), Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT), kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Kanban statuses, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis), Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (KANBAN_TYPES subset, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Thought-type color tokens, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, Thread extraction)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Permanent file write discipline)
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Interview stages, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Thread extraction, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Thread extraction, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Thread extraction, _enrichment_status)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Thread extraction, Two-Prompt Extraction Sequence)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Speaker Consolidation)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Thread extraction)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Noise filtering, Panning / Gold-Found)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output modes (file / entity-metadata / thought), Permanent file write discipline)
- **[recipes/google-activity-import](../../recipes/google-activity-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Panning / Gold-Found, Per-category noise filtering)
- **[recipes/grok-export-import](../../recipes/grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Speaker Consolidation, Transcript assembly)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Permanent file write discipline)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Panning / Gold-Found)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Thread extraction)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Topic shift detection)
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, Secret scanning)
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Two-sheet import (Conversations + Memory))
- **[recipes/repo-learning-coach/server](../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (Artifact kinds (takeaway, confusion, summary), Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (LearningArtifactKind, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Panning / Gold-Found, Skip rules)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Thread extraction, Three-pass person resolution)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Thread extraction)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType taxonomy, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ThoughtType, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[recipes/wiki-compiler](../../recipes/wiki-compiler/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Permanent file write discipline)
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread eligibility gate)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Speaker Consolidation)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Thread extraction)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Thought-entity mention role and evidence, Thread extraction)
- **[skills](../CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Permanent file write discipline)
- **[skills/claudeception](../claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Thread extraction)
- **[skills/deal-memo-drafting](../deal-memo-drafting/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Conviction state, Decision-readiness, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Structural risk vs. business risk, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/heavy-file-ingestion](../heavy-file-ingestion/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output directory convention (.ob1/), Permanent file write discipline)
- **[skills/heavy-file-ingestion/scripts](../heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Permanent file write discipline)
- **[skills/weekly-signal-diff](../weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Panning / Gold-Found, Signal diff vs digest)
- **[skills/work-operating-model](../work-operating-model/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Five fixed interview layers, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/world-model-diagnostic](../world-model-diagnostic/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, Simulated judgment)
