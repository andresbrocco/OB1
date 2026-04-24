# CONTEXT.md — Recipes

## Purpose

Standalone capability builds that add a new feature or workflow to an Open Brain instance. Each recipe is a self-contained contribution that extends the system without belonging to the structured learning path. Examples include data importers (ChatGPT, Gmail, Obsidian, Instagram), deduplication utilities, AI skill packs, knowledge-graph generators, and capture pipeline enhancements.

## Responsibility Boundaries

- **Owns**: The implementation files, `README.md`, and `metadata.json` for each discrete capability build
- **Delegates to**: The core Open Brain Supabase database (the `thoughts` table and any schema extensions), external services declared in `metadata.json` `requires.services`, and reusable behaviors from `skills/` via `requires_skills`
- **Does not handle**: Curated progressive learning sequences (that is `extensions/`), reusable concept guides referenced by multiple contributions (that is `primitives/`), pure prompt/skill packs with no build step (that is `skills/`), database schema definitions shared across contributions (that is `schemas/`), frontend templates (that is `dashboards/`), or MCP server wiring (that is `integrations/`)

## Key Concepts

- **Recipe vs. Skill**: A recipe requires a build step — SQL migrations, edge functions, scripts, config files. A skill is a plain-text prompt/behavior file with no infrastructure setup. Tightly coupled glue behavior may live as a `*.skill.md` inside a recipe folder, but reusable behaviors should live canonically in `skills/` and be declared with `requires_skills`.
- **Recipe vs. Extension**: Extensions are curated, ordered, and form a progressive learning path. Recipes are unordered, standalone, and open to community contribution without maintainer pre-approval.
- **`_template/`**: The `_template` subdirectory contains only a `metadata.json` skeleton — it is not a real recipe and should not be documented or indexed as one.
- **`metadata.json` contract**: Each recipe subfolder must have `metadata.json` declaring `name`, `description`, `category: "recipes"`, `author`, `version`, `requires` (with `open_brain`, `services`, `tools`), `tags`, `difficulty`, and `estimated_time`. Automated PR review validates this schema.

## Non-Obvious Details

- Recipes that include a `*.skill.md` file alongside code are using a local skill coupling pattern, not the canonical reusable-skill pattern. If the behavior is useful across multiple recipes, it should be moved to `skills/` and referenced via `requires_skills` in `metadata.json`.
- Some recipe folders contain both a schema migration (`schema.sql`) and runtime code — the migration is the contributor's responsibility to document and test against a live instance; the core `thoughts` table must not be altered.
- The `fingerprint-dedup-backfill` recipe includes a destructive helper (`delete-duplicates.mjs`). This is intentional and scoped to deduplication logic, not a violation of the no-`DELETE` guard (which applies to unqualified bulk deletes in SQL files).

## Related Modules

- **[.claude/skills](../.claude/skills/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Recipe vs. Extension distinction, Recipe vs. Skill distinction, Skill file format, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[extensions](../extensions/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (5-file extension contract, Learning path (learning_order 1-6), Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[integrations](../integrations/CONTEXT.md)** — Depends on for External connection points that bring data into Open Brain or replace its infrastructure layer, including message-capture bots, async LLM queue workers, and a Kubernetes self-hosted deployment variant
- **[primitives](../primitives/CONTEXT.md)** — Depends on for Maintainer-curated reusable concept guides for foundational patterns shared across two or more extensions
- **[recipes/claudeception](claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Recipe vs. Extension distinction, Recipe vs. Skill distinction, Skill Lifecycle, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/entity-wiki](entity-wiki/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Output modes (file / entity-metadata / thought), Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/infographic-generator](infographic-generator/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompts file format, Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/research-to-decision-workflow](research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Prompt stubs, Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[schemas](../schemas/CONTEXT.md)** — Depends on for Community-contributed idempotent SQL migrations that extend the Open Brain Supabase database with knowledge-graph tables, enrichment columns, RPCs, triggers, and RLS policies
- **[skills](../skills/CONTEXT.md)** — Depends on for Reusable AI behavioral protocols — prompt packs and process definitions that govern how an AI client handles a specific class of recurring problem
- **[skills/claudeception](../skills/claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Recipe vs. Extension distinction, Recipe vs. Skill distinction, Skill lifecycle (creation to archival), _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[skills/work-operating-model](../skills/work-operating-model/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
