# CONTEXT.md — N Agentic Harnesses

## Purpose

Provides a reusable AI skill pack for designing, auditing, and improving the harness layer of agentic products. The skill targets the structural layer surrounding an AI model — tool boundaries, permission policy, workflow state, context assembly, evaluation loops, and operator visibility — rather than the model's reasoning capabilities themselves.

## Responsibility Boundaries

- **Owns**: Skill routing logic (SKILL.md as the index), all reference content covering harness primitives and playbooks, decision posture for solo-maintainable agentic systems
- **Delegates to**: The consuming AI client, which reads SKILL.md as a prompt and selectively loads reference files per request classification
- **Does not handle**: Model selection, prompt engineering for task-specific domains, infrastructure provisioning, or database schema

## Key Concepts

- **Harness**: The structural layer around an AI model — tools, permissions, state management, context assembly, evaluation, and observability — as distinct from the model itself
- **Harness primitives**: Discrete, recombinable building blocks (tool registries, approval gates, session stores, eval suites) that together define a harness's capability
- **Product shape**: A classification of the system being built (code agent, chat assistant, workflow orchestrator, internal copilot, embedded feature, hybrid) used to route advice
- **Mode classification**: The skill distinguishes `design` (new build or major rebuild), `evaluation` (gap analysis of an existing harness), and `design + evaluation` (both) before loading reference files
- **Approval gates**: Explicit human-in-the-loop checkpoints in a tool-use pipeline, treated as a first-class architectural concern rather than an optional add-on

## Non-Obvious Details

- SKILL.md is both the entry point and the complete routing index. It intentionally discourages reference-to-reference chains ("Do not rely on reference-to-reference chains. This file is the index."), so all cross-cutting concerns must be resolved at the SKILL.md level before loading individual references.
- Reference files are numbered (`01`–`11`) and designed to be read selectively per request, not as a linear curriculum. Loading unnecessary references is considered a failure mode.
- The skill has an explicit solo-developer bias: multi-agent coordination is not recommended by default and requires justification.
- `references/11-codex-translation-notes.md` exists solely for adapting the skill to Codex-specific environments and is not part of normal usage.
- Evaluation output format differs structurally from design output: evaluation leads with findings ordered by severity, while design leads with recommended shape and MVP boundary.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Approval gates)
- **[.github](../../.github/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Artifact handoff between workflows, Harness, Harness primitives)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, details JSONB freeform metadata field)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, Product shape)
- **[integrations](../../integrations/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, Shared config with sensitivity tiers)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic))
- **[primitives](../../primitives/CONTEXT.md)** — Shares PR Review and CI Automation domain (Approval gates, Curation gate)
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Two-phase pipeline)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Mode classification)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Approval gates, Harness, Harness primitives, Retrospective Mode)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Two-phase pipeline)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Self-improvement protocol)
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares PR Review and CI Automation domain (Approval gates, Secret scanning)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Product shape)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Schema-aware routing)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, Type backfill (metadata.type promotion))
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Mode classification)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Parallel capture pipeline)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, ThoughtMetadata)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Product shape, ThoughtMetadata)
- **[recipes/wiki-compiler](../../recipes/wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Phase toggles)
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Synthesizer catalogue)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Synthesizer catalogue (plugin map))
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Async queue with content-addressed re-queue, Harness, Harness primitives)
- **[skills/claudeception](../claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Approval gates, Harness, Harness primitives, Retrospective mode)
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, Product shape)
- **[skills/heavy-file-ingestion](../heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Mode classification)
