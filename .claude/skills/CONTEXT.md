# CONTEXT.md — Skills

## Purpose

Houses Claude Code skill definitions for OB1 maintainer workflows. Skills are reusable agent prompt packs invoked by name (e.g., "review PR 21") and automate multi-step admin tasks that go beyond what CI can handle.

## Responsibility Boundaries

- **Owns**: Skill definition files (`.md` with YAML frontmatter) that encode repeatable admin workflows for Claude Code
- **Delegates to**: The CI workflow (`.github/workflows/ob1-review.yml`) for mechanical rule checks; `gh` CLI for GitHub API interactions
- **Does not handle**: Automated CI checks, actual posting to GitHub without user confirmation, or any write operations without explicit user approval

## Key Concepts

**Skill file format**: Each `.md` file uses YAML frontmatter (`name`, `description`) and a structured markdown body that serves as the agent's instruction set. Claude Code discovers and invokes these by name.

**Human-judgment layer**: Skills in this directory are specifically designed to augment CI automation — they handle the subjective and contextual checks (mission fit, security depth, naming consistency) that automated rules cannot reliably perform.

## Non-Obvious Details

- Skills always end with an explicit user confirmation step before side-effecting operations (e.g., posting a PR comment). This is by design — the outputs are artifacts for human review, not autonomous actions.
- The `review-pr` skill checks out the PR branch locally via `gh pr checkout` and is responsible for returning to the original branch when done (`git checkout <branch>`).
- Admin summary and Discord draft outputs from `review-pr` are formatted for sharing with admins who do not have Claude Code — they are designed as standalone artifacts, not internal notes.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Docs PR bypass, Idempotent PR comment via ob1-automated-review marker, Trusted contributor auto-review, Two-stage review pipeline (deterministic gate + LLM qualitative review), security-blocked and needs-maintainer-triage label lifecycle)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Idempotent PR comments, Two-stage review pipeline, pull_request vs pull_request_target security split)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Skill file format)
- **[primitives](../../primitives/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Curation gate)
- **[recipes](../../recipes/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Recipe vs. Extension distinction, Recipe vs. Skill distinction, Skill file format, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Skill Lifecycle, Skill file format)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Prompt injection defense)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompts file format, Skill file format)
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Secret scanning)
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Skill file format)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Prompt injection defense)
- **[skills](../../skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (SKILL.md, Skill file format)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Skill file format, Skill lifecycle (creation to archival))
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Approval gates)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Mary's Law Check)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Skill file format)
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Simulated judgment)
