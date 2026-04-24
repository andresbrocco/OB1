# CONTEXT.md — Research-to-Decision Workflow

## Purpose

Provides a composable workflow scaffold for turning research inputs (sources, meetings, financial models) into decision-ready artifacts. It is not a standalone capability but an orchestration pattern that sequences five named OB1 skills into either an operator decision pipeline or an investor diligence pipeline.

## Responsibility Boundaries

- **Owns**: The ordering, handoff logic, and branching between skills; the workspace folder structure; prompt stubs that invoke each skill in sequence
- **Delegates to**: `competitive-analysis`, `financial-model-review`, `deal-memo-drafting`, `research-synthesis`, and `meeting-synthesis` skills for the actual analytical work
- **Does not handle**: The content of any individual analysis step; Open Brain capture mechanics (those are optional add-ons noted at the end of the template)

## Key Concepts

- **Operator path**: A shorter pipeline (brief → competitive analysis → research synthesis → meeting synthesis) used when no financial model or investment memo is needed.
- **Investor path**: The full pipeline that adds financial model review and deal memo drafting after competitive analysis, ending in a structured recommendation memo.
- **Handoff checklist**: A table that formalizes what each step consumes and produces, and the "ready when" condition that gates the next step. This is the primary mechanism for knowing when to advance.
- **Prompt stubs**: Minimal natural-language invocations that tell an AI client which skill to run and which prior output files to include as context. They are the executable units of the workflow.

## Non-Obvious Details

- Steps are individually skippable via explicit skip rules (no model artifact → skip model review; no meeting → skip meeting synthesis; final artifact is a brief, not a memo → skip deal memo). The skip rules are defined in the template, not inferred at runtime.
- The `00-brief.md` file drives audience and path selection at the start and is re-referenced by later steps (financial model review, deal memo drafting) as decision context. It is not a one-time input.
- Open Brain capture moments are optional and listed explicitly at the end of the template; they are not wired into the handoff checklist and must be triggered manually.
- This recipe has no code, no SQL, and no deployable artifact. Its sole deliverable is `workflow-template.md`, a markdown scaffold intended to be copied into a working project.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Skill file format)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Prompt stubs)
- **[recipes](../CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal-based filtering, Skip rules)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Skill Lifecycle)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Noise filtering, Skip rules)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Output modes (file / entity-metadata / thought), Prompt stubs)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Skip rules)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Prompts file format)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Skip rules)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Gold-Found, Panning, Skip rules)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, Skip rules)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Skip rules, Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Skip rules, Thread eligibility gate)
- **[skills](../../skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, SKILL.md)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Prompt stubs, Skill lifecycle (creation to archival))
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Depends on for AI skill for drafting structured deal, IC, partnership, or acquisition memos from pre-existing diligence materials
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Depends on for Investor-first AI skill pack for reviewing existing financial models, forecasts, and scenario sets for assumption quality, structural risk, and decision usefulness
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Panning / Gold-Found, Skip rules)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Skip rules)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Investor path)
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Firm finding / Inference / Open question labeling, Five-principle evaluation, Investor path)
