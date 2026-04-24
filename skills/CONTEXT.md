# CONTEXT.md — Skills

## Purpose

Skills are reusable AI behavioral protocols — prompt packs and process definitions that tell an AI client how to handle a specific class of recurring problem. They are not code libraries or plugins; they are structured instructions that get loaded into an AI's context to govern its behavior for a session or task.

## Responsibility Boundaries

- **Owns**: Defining trigger conditions, step-by-step process, output contracts, and guard rails for a named AI behavior
- **Delegates to**: Recipes (for workflow orchestration that composes multiple skills), integrations (for data capture and retrieval), and the AI client runtime for actual execution
- **Does not handle**: Deployment infrastructure, database schema, or MCP server configuration — those live in integrations and primitives

## Key Concepts

**SKILL.md**: The canonical artifact for each skill. Contains a YAML front matter block (name, description, author, version) followed by structured sections: Problem, Trigger Conditions, Process, Output, and Notes. The description field is written to be consumed by an AI routing layer, not just humans.

**Trigger Conditions**: Explicit phrases, symptoms, or contextual signals that tell the AI when to invoke a skill. Skills are pulled by trigger, not pushed by schedule — they fire when the AI recognizes a matching condition, not automatically.

**Variants**: Some skills (e.g., `heavy-file-ingestion`, `n-agentic-harnesses`) include a `variants/` subdirectory with client-specific versions (claude-desktop, claude-code, codex, anthropic). The root SKILL.md is the canonical version; variants adapt for a specific AI client's tool surface.

**References**: Some skills carry a `references/` subdirectory with supporting knowledge documents the skill SKILL.md instructs the AI to load selectively, rather than embedding all content inline. This pattern keeps the root SKILL.md as a lightweight index and avoids unnecessary token use.

**Lessons Log**: Skills that evolve from real use (e.g., `panning-for-gold`) embed a production lessons log directly in the SKILL.md. Rules in the "Critical Rules" sections reflect failures that have already occurred, not hypothetical edge cases.

## Non-Obvious Details

- Skills that reference Open Brain tools (capture, search) do not hard-code tool names. The MCP connector prefix varies by environment, so SKILL.md files use soft references like "the available capture tool (often `capture_thought`; prefixes vary by connector)."
- The `_template/` subdirectory is a scaffolding starting point, not a runtime skill. It ships a minimal SKILL.md and metadata.json to standardize new contributions.
- Skills are intentionally reusable across AI clients. Anything client-specific belongs in a variant, not the root SKILL.md.
- `panning-for-gold` includes a self-improvement loop (Phase 4) where the skill instructs the AI to update the SKILL.md file itself after each session. This means the file on disk is a living artifact that accumulates production learnings.
- `heavy-file-ingestion` is the only skill that ships executable scripts (`scripts/`), making it a hybrid of behavioral protocol and tool implementation.

## Related Modules

- **[.claude/skills](../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (SKILL.md, Skill file format)
- **[dashboards](../dashboards/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (MCP JSON-RPC proxy connection pattern (SvelteKit dashboard), REST API connection pattern (Next.js dashboard), Variants, iron-session cookie auth)
- **[dashboards/open-brain-dashboard](../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SSR auth, Variants)
- **[dashboards/open-brain-dashboard-next](../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Variants, iron-session cookie auth)
- **[dashboards/open-brain-dashboard-next/components](../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Reflection types (decision_trace, lesson_trace, retrospective, hypothesis))
- **[dashboards/open-brain-dashboard/src](../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (SvelteKit locals augmentation, Variants)
- **[dashboards/open-brain-dashboard/src/routes](../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Auth guard via layout.server.ts, Variants, latestResultKey highlight)
- **[docs](../docs/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Progressive onboarding sequence)
- **[extensions](../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Output Contract)
- **[integrations](../integrations/CONTEXT.md)** — Depends on for External connection points that bring data into Open Brain or replace its infrastructure layer, including message-capture bots, async LLM queue workers, and a Kubernetes self-hosted deployment variant
- **[recipes](../recipes/CONTEXT.md)** — Provides auto-capture, autodream-brain-sync, claudeception, ... consumed by this module
- **[recipes/chatgpt-conversation-import](../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Output Contract, Pyramid summaries)
- **[recipes/claudeception](../recipes/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Retrospective Mode)
- **[recipes/entity-wiki](../recipes/entity-wiki/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Output modes (file / entity-metadata / thought))
- **[recipes/infographic-generator](../recipes/infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Output Contract)
- **[recipes/obsidian-vault-import](../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Output Contract, Two-tier chunking (heading split + LLM fallback))
- **[recipes/repo-learning-coach](../recipes/repo-learning-coach/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, Lessons Log, RepoLearningConfig)
- **[recipes/repo-learning-coach/server](../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Lessons Log)
- **[recipes/repo-learning-coach/src](../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Lessons Log)
- **[recipes/repo-learning-coach/src/lib](../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Lessons Log)
- **[recipes/research-to-decision-workflow](../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, SKILL.md)
- **[recipes/wiki-compiler](../recipes/wiki-compiler/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Output Contract)
- **[recipes/wiki-synthesis](../recipes/wiki-synthesis/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Resume-safe JSONL state log)
- **[skills/claudeception](claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Retrospective mode)
- **[skills/heavy-file-ingestion](heavy-file-ingestion/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Client variants and build exports, Variants)
- **[skills/heavy-file-ingestion/scripts](heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Output Contract)
- **[skills/panning-for-gold](panning-for-gold/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Permanent file write discipline)
- **[skills/work-operating-model](work-operating-model/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, SKILL.md)
