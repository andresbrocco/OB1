# CONTEXT.md — .github

## Purpose

Defines the automated contribution governance layer for the Open Brain repo. This includes the PR review pipeline, CI gate rules, issue triage, metadata schema validation, and community automation workflows. It enforces contribution quality and security without requiring manual maintainer review of every incoming PR.

## Responsibility Boundaries

- **Owns**: All GitHub Actions workflows, the canonical `metadata.schema.json` that all contributions are validated against, issue templates, the PR template, and markdownlint configuration.
- **Delegates to**: `CONTRIBUTING.md` for human-readable contribution rules; `docs/` for setup guides referenced in gate failure messages.
- **Does not handle**: Content-level business logic for any contribution category (recipes, schemas, etc.). The gate enforces structural and safety rules only.

## Key Concepts

**Two-stage review pipeline**: The `ob1-gate.yml` workflow runs a deterministic shell-based gate on every PR and uploads a structured artifact (`ob1-pr-gate-context`). The `ob1-pr-followups.yml` workflow triggers on gate completion, downloads this artifact, posts the gate summary as an idempotent PR comment, and conditionally invokes Claude (via `claude-code-action`) for qualitative review. The two stages are intentionally decoupled so the gate can fail fast without invoking LLM credits.

**Trusted contributor auto-review**: Claude auto-review in `ob1-pr-followups.yml` only fires when the gate passes AND the author is classified as `OWNER`, `MEMBER`, `COLLABORATOR`, or `CONTRIBUTOR` by GitHub AND the author has fewer than 3 open PRs. Untrusted or over-quota contributors require a maintainer to manually dispatch `claude-review.yml`.

**Docs PRs bypass contribution checks**: The gate detects `[docs]`-prefixed PR titles or PRs that touch no contribution directories and skips all structural checks. This avoids false failures on governance-only changes.

## Non-Obvious Details

- The gate comment is idempotent: `ob1-pr-followups.yml` searches for an existing bot comment containing the `<!-- ob1-automated-review -->` marker and updates it rather than creating a new comment on each push.
- `security-blocked` and `needs-maintainer-triage` labels are managed programmatically by `ob1-pr-followups.yml` and will be removed automatically when the triggering condition is resolved on a new push.
- Rule 11 (LLM clarity review) is a no-op placeholder in `ob1-gate.yml`; it always passes. The actual LLM review runs separately in `ob1-pr-followups.yml` to keep the gate deterministic and fast.
- The `claude-review.yml` workflow is `workflow_dispatch`-only and accepts an `extra_focus` string, allowing maintainers to direct Claude's attention to a specific concern without re-running the full gate.
- Rule 14 (remote MCP pattern) scans `.md`, `.ts`, and `.js` files for `claude_desktop_config`, `StdioServerTransport`, and `mcpServers.*command` to block the legacy local-server pattern.
- PR quota (`over_quota`) is evaluated against currently open PRs for the author login at the moment the gate artifact is processed, not at PR open time.

## Related Modules

- **[.claude/skills](../.claude/skills/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Docs PR bypass, Idempotent PR comment via ob1-automated-review marker, Trusted contributor auto-review, Two-stage review pipeline (deterministic gate + LLM qualitative review), security-blocked and needs-maintainer-triage label lifecycle)
- **[.github/workflows](workflows/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, PR quota enforcement)
- **[integrations](../integrations/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, PR quota enforcement)
- **[integrations/entity-extraction-worker](../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (ExtractionCostCapError, PR quota enforcement, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[primitives](../primitives/CONTEXT.md)** — Shares PR Review and CI Automation domain (Curation gate, Docs PR bypass, Idempotent PR comment via ob1-automated-review marker, Trusted contributor auto-review, Two-stage review pipeline (deterministic gate + LLM qualitative review), security-blocked and needs-maintainer-triage label lifecycle)
- **[recipes/chatgpt-conversation-import](../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Sync log)
- **[recipes/email-history-import](../recipes/email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Sync log, Two-layer dedup)
- **[recipes/entity-wiki](../recipes/entity-wiki/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, security-blocked and needs-maintainer-triage label lifecycle)
- **[recipes/fingerprint-dedup-backfill](../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Idempotent PR comment via ob1-automated-review marker)
- **[recipes/google-activity-import](../recipes/google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Idempotent PR comment via ob1-automated-review marker)
- **[recipes/grok-export-import](../recipes/grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Idempotent PR comment via ob1-automated-review marker)
- **[recipes/instagram-import](../recipes/instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Idempotent PR comment via ob1-automated-review marker)
- **[recipes/journals-blogger-import](../recipes/journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Idempotent PR comment via ob1-automated-review marker)
- **[recipes/life-engine](../recipes/life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Idempotent PR comment via ob1-automated-review marker)
- **[recipes/live-retrieval](../recipes/live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Session-scoped deduplication)
- **[recipes/obsidian-vault-import](../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Idempotent PR comment via ob1-automated-review marker)
- **[recipes/perplexity-conversation-import](../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Local sync log deduplication)
- **[recipes/schema-aware-routing](../recipes/schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[recipes/thought-enrichment](../recipes/thought-enrichment/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[recipes/typed-edge-classifier](../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Hard cost cap with proactive parallelism clamping, PR quota enforcement)
- **[recipes/vercel-neon-telegram](../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (In-memory rate limiter with cold-start reset, PR quota enforcement)
- **[recipes/vercel-neon-telegram/src](../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (PR quota enforcement, in-memory sliding-window rate limiter)
- **[recipes/wiki-synthesis](../recipes/wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[recipes/wiki-synthesis/scripts](../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue (plugin map), Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[recipes/x-twitter-import](../recipes/x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Idempotent PR comment via ob1-automated-review marker)
- **[schemas/enhanced-thoughts](../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, idempotent schema migration)
- **[skills/claudeception](../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Open Brain deduplication workflow)
- **[skills/heavy-file-ingestion](../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Cost tier escalation, PR quota enforcement)
- **[skills/n-agentic-harnesses](../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[skills/panning-for-gold](../skills/panning-for-gold/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, security-blocked and needs-maintainer-triage label lifecycle)
- **[skills/world-model-diagnostic](../skills/world-model-diagnostic/CONTEXT.md)** — Shares Prompt Injection and Security domain (Simulated judgment, security-blocked and needs-maintainer-triage label lifecycle)
