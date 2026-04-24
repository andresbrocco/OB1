# CONTEXT.md — Panning for Gold

## Purpose

Transforms unstructured raw captures (voice transcripts, brain dumps, stream-of-consciousness notes) into evaluated, prioritized idea inventories stored in Open Brain. The recipe operates as a four-phase skill file (`panning-for-gold.skill.md`) that an AI agent follows, not as runnable code.

## Responsibility Boundaries

- **Owns**: The full extraction-to-capture pipeline for multi-topic raw input: saving raw input, cleaning speaker attribution, extracting idea threads, evaluating top threads, synthesizing a gold-found file, and capturing results to Open Brain.
- **Delegates to**: Background evaluator agents (Opus/Sonnet/Haiku via OpenRouter) for per-thread deep evaluation; Open Brain's `capture_thought` and `search_thoughts` MCP tools for storage and retrieval; the Auto-Capture Protocol recipe (if installed) for session-end summaries.
- **Does not handle**: Structured notes, single-topic documents, or any input where the user's intent is already clear. Those don't need thread extraction.

## Key Concepts

- **Panning / Gold-Found**: The metaphor for the process. "Panning" is exhaustive extraction without filtering; "gold-found" is the synthesis document produced at the end.
- **Thread**: A single extracted idea, observation, or signal from the raw input. Threads are intentionally granular — related but distinct ideas (e.g., two different uses of the same product) are kept separate because they have different evaluations.
- **ACT NOW / RESEARCH MORE / PARK / KILL**: The four triage verdicts used in Phase 2 evaluation. Only 3–5 threads should ever reach ACT NOW status per session.
- **Speaker Consolidation (Phase 0.5)**: A pre-extraction step specific to multi-speaker voice transcripts. Auto-generated speaker labels from tools like Otter or Plaud are treated as actively misleading, not just unreliable, because environment changes cause the same person to receive multiple labels and different people to share a label. Attribution uses anchor lines and scene-based reasoning rather than trusting the label numbers.
- **Flywheel closure**: The final capture step (Phase 3.5) is what connects the panning output back into Open Brain, making extracted ideas discoverable in future sessions.

## Non-Obvious Details

- The skill file is self-improving: a Lessons Log section records production failures and the rules that resulted from them. The Critical Rules section was built entirely from real mistakes (agents lost to context compaction, full transcript re-reads burning 30K tokens, uninverted speaker attribution corrupting 40+ threads).
- Synthesis (Phase 3) is explicitly written by the orchestrating agent inline, never delegated to a sub-agent. This rule exists because sub-agents disappear across compaction boundaries and their return values cannot be relied upon.
- Phase 1 defaults to over-extraction: 80+ threads for a one-hour conversation is described as normal. Curation happens in Phase 2 triage, not during extraction. Under-extraction on the first pass is listed as the single most common failure mode.
- The token-efficiency strategy (summary-first, Grep for quotes, read first/last 50 lines only) is a first-class concern, not an optimization. Summaries-first was added after a single session burned ~30K tokens on unnecessary re-reads.
- The recipe cross-references the Auto-Capture Protocol recipe but remains independently functional if that recipe is not installed.

## Related Modules

- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Kanban workflow (task/idea types only))
- **[dashboards/open-brain-dashboard](../../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, ThoughtType)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, kanban workflow (task/idea types))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Kanban statuses)
- **[dashboards/open-brain-dashboard-next/components](../../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, KANBAN_TYPES eligibility boundary, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, KANBAN_TYPES subset)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Thought-type color tokens)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, ThoughtType)
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Interview stages)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Operating Model Layers)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Speaker Consolidation, Thread)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Gold-Found, Noise filtering, Panning)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Emergent cached view, Flywheel closure)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Gold-Found, High-value categories, Panning, Per-category noise filtering)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Speaker Consolidation, Thread, Transcript assembly)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Gold-Found, Panning)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Compaction-safe persistence, user_id as channel chat_id)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Topic shift detection)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Two-sheet import (Conversations + Memory))
- **[recipes/repo-learning-coach](../repo-learning-coach/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Flywheel closure, Understanding State)
- **[recipes/repo-learning-coach/server](../repo-learning-coach/server/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, Artifact kinds (takeaway, confusion, summary))
- **[recipes/repo-learning-coach/src](../repo-learning-coach/src/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, LearningArtifactKind)
- **[recipes/repo-learning-coach/src/lib](../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, LearningArtifactKind)
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Gold-Found, Panning, Skip rules)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, Gold-Found, Panning)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Compaction-safe persistence, In-memory rate limiter with cold-start reset)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Compaction-safe persistence, Telegram webhook singleton bot, in-memory sliding-window rate limiter)
- **[recipes/vercel-neon-telegram/src/app/api](../vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Bot singleton pattern, Compaction-safe persistence, Telegram webhook secret authentication)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Thought Types and Taxonomy domain (ACT NOW / RESEARCH MORE / PARK / KILL, ThoughtType)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Thread eligibility gate)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Speaker Consolidation, Thread)
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Conviction state, Decision-readiness)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Structural risk vs. business risk)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Gold-Found, Panning, Signal diff vs digest)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Five fixed interview layers)
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Flywheel closure, OB1-connected mode vs. direct-chat mode, World-model paradigm (vector database / structured ontology / signal-fidelity))
