# CONTEXT.md — Primitives

## Purpose

Reusable concept guides that document foundational technical patterns shared across two or more extensions. Each primitive is a standalone explainer — not code to run, but a guide a developer or AI agent follows to implement a cross-cutting concern correctly.

## Responsibility Boundaries

- **Owns**: Canonical documentation of shared infrastructure patterns (deployment, connectivity, security, debugging)
- **Delegates to**: Extensions and recipes that implement the actual feature logic built on top of these patterns
- **Does not handle**: Feature-specific logic, data schemas, or AI skills — those belong in their respective top-level categories

## Key Concepts

- **Primitive**: A reusable concept guide that must be referenced by at least two extensions before it is accepted into this directory. This cross-reference requirement distinguishes primitives from one-off documentation.
- **Curation gate**: This directory is maintainer-curated. Community contributors cannot add primitives without maintainer approval, unlike `recipes/`, `schemas/`, or `skills/` which are open for community contributions.
- **Concept vs. code**: Primitives contain guides and explanations, not executable code. They describe patterns (e.g., how RLS works, how to wire a remote MCP connection) that are then applied inside extension folders.

## Non-Obvious Details

- The `_template/` subfolder provides the `metadata.json` scaffold for new primitives but does not itself represent a valid primitive.
- The requirement that a concept must be referenced by 2+ extensions before graduating to a primitive means primitives are extracted upward from existing extensions, not authored speculatively.
- All MCP-related primitives assume Supabase Edge Functions as the deployment target; local transport (stdio, local Node.js) is explicitly out of scope per project guard rails.

## Related Modules

- **[.claude/skills](../.claude/skills/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Curation gate)
- **[.github](../.github/CONTEXT.md)** — Shares PR Review and CI Automation domain (Curation gate, Docs PR bypass, Idempotent PR comment via ob1-automated-review marker, Trusted contributor auto-review, Two-stage review pipeline (deterministic gate + LLM qualitative review), security-blocked and needs-maintainer-triage label lifecycle)
- **[.github/workflows](../.github/workflows/CONTEXT.md)** — Shares PR Review and CI Automation domain (Curation gate, Idempotent PR comments, Two-stage review pipeline, pull_request vs pull_request_target security split)
- **[recipes](../recipes/CONTEXT.md)** — Provides deploy-edge-function, remote-mcp, rls, ... consumed by this module
- **[recipes/obsidian-vault-import](../recipes/obsidian-vault-import/CONTEXT.md)** — Shares PR Review and CI Automation domain (Curation gate, Secret scanning)
- **[skills/n-agentic-harnesses](../skills/n-agentic-harnesses/CONTEXT.md)** — Shares PR Review and CI Automation domain (Approval gates, Curation gate)
