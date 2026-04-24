# CONTEXT.md — Workflows

## Purpose

Automates the full contribution lifecycle for the OB1 community repo: structural validation of PRs, AI-assisted quality review, issue triage, contributor onboarding, labeling, and release management. The pipeline enforces contribution standards without requiring manual maintainer attention for mechanical checks.

## Responsibility Boundaries

- **Owns**: All GitHub Actions automation — PR gating, LLM review invocation, issue triage, contributor trust policy, labeling, Discord announcements, markdown linting, and release drafting
- **Delegates to**: `anthropics/claude-code-action` for AI review judgments; `release-drafter/release-drafter` for changelog generation; `.github/metadata.schema.json` for metadata validation rules
- **Does not handle**: Merge authorization (branch protection handles that); the actual content standards (defined in `CONTRIBUTING.md` and `CLAUDE.md`)

## Key Concepts

**Two-stage review pipeline**: The PR gate (`ob1-gate.yml`) runs deterministic shell-script checks and uploads a structured artifact. The follow-up workflow (`ob1-pr-followups.yml`) triggers on `workflow_run` completion, downloads that artifact, posts the gate summary as a PR comment, and — conditionally — invokes Claude for qualitative review. The two stages are deliberately decoupled so the gate can remain fast and the LLM review only runs when the gate passes.

**Artifact handoff**: `ob1-gate.yml` writes `ob1-review-context.json` and `ob1-review-summary.md` into a retained artifact (`ob1-pr-gate-context`). The follow-up workflow downloads this artifact rather than re-running the checks, preserving a consistent source of truth and avoiding duplicate GitHub API calls.

**Trust and quota policy**: `ob1-pr-followups.yml` classifies contributors as trusted (`OWNER`, `MEMBER`, `COLLABORATOR`, `CONTRIBUTOR` association) and checks if they have more than 3 open PRs. The Claude auto-review only fires for trusted contributors who are under quota and whose gate passed cleanly. Untrusted or over-quota PRs still get the gate summary comment and a `needs-maintainer-triage` label.

**`claude-review.yml` is manually dispatched**: Unlike the automatic follow-up, this workflow is triggered via `workflow_dispatch` (e.g., by a maintainer). It uses a broader tool allowlist and is intended for PRs that need a second look or for contributors not covered by the auto-review path.

**Idempotent PR comments**: The follow-up workflow uses an HTML marker (`<!-- ob1-automated-review -->`) to find and update an existing bot comment rather than posting a new one on each re-run.

## Non-Obvious Details

- `ob1-gate.yml` uses `pull_request` (not `pull_request_target`), which means it checks out the PR head ref directly. The gate has `contents: read` only and does not write back to GitHub — all writes are deferred to the follow-up workflow, which checks out the default branch instead. This split avoids the security risk of running untrusted code with write permissions.
- Rules are numbered non-consecutively (1–11, 14, 15, then 12–13). Rules 12–13 (scope check, internal link validation) appear after the numbered sequence because they were added later; rule numbers were preserved to avoid breaking any external references.
- The `secret_blocked` flag causes an immediate `security-blocked` label regardless of overall gate pass/fail status. A credential match blocks the label from being removed even if the contributor pushes a fix, until the gate re-runs clean.
- `auto-label.yml` uses `pull_request_target` (safe for labeling since it only reads file paths, not file contents) while `ob1-gate.yml` uses `pull_request`. This difference is intentional.
- The `ob1-gate.yml` detects docs-only PRs either by `[docs]` title prefix or by the absence of any recognized contribution directory in the changed files, and skips contribution checks entirely in that case.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares PR Review and CI Automation domain (Admin review vs CI automation, Idempotent PR comments, Two-stage review pipeline, pull_request vs pull_request_target security split)
- **[.github](../CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, PR quota enforcement)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Async worker with cost cap, Contributor trust and quota policy)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, ExtractionCostCapError, wall-clock budget)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Multi-provider LLM fallback (OpenRouter > OpenAI > Anthropic), Two-stage review pipeline)
- **[primitives](../../primitives/CONTEXT.md)** — Shares PR Review and CI Automation domain (Curation gate, Idempotent PR comments, Two-stage review pipeline, pull_request vs pull_request_target security split)
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Two-phase pipeline)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, Sync log)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Artifact handoff between workflows, Retrospective Mode)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, Sync log, Two-layer dedup)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, pull_request vs pull_request_target security split)
- **[recipes/fingerprint-dedup-backfill](../../recipes/fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Idempotent PR comments)
- **[recipes/google-activity-import](../../recipes/google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Idempotent PR comments)
- **[recipes/grok-export-import](../../recipes/grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Idempotent PR comments)
- **[recipes/infographic-generator](../../recipes/infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Two-phase pipeline)
- **[recipes/instagram-import](../../recipes/instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Idempotent PR comments)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Idempotent PR comments)
- **[recipes/life-engine](../../recipes/life-engine/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Self-improvement protocol)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, Session-scoped deduplication)
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Idempotent PR comments)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, Local sync log deduplication)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Two-stage review pipeline)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (LLM classification prompt with importance/confidence calibration, Two-stage review pipeline)
- **[recipes/typed-edge-classifier](../../recipes/typed-edge-classifier/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, Hard cost cap with proactive parallelism clamping)
- **[recipes/vercel-neon-telegram](../../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Parallel capture pipeline)
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, in-memory sliding-window rate limiter)
- **[recipes/wiki-compiler](../../recipes/wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Phase toggles)
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue, Two-stage review pipeline)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Synthesizer catalogue (plugin map), Two-stage review pipeline)
- **[recipes/x-twitter-import](../../recipes/x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Idempotent PR comments)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, idempotent schema migration)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Async queue with content-addressed re-queue)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Artifact handoff between workflows, Retrospective mode)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Cost Management and Rate Limiting domain (Contributor trust and quota policy, Cost tier escalation)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Artifact handoff between workflows, Harness, Harness primitives)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, pull_request vs pull_request_target security split)
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Prompt Injection and Security domain (Simulated judgment, pull_request vs pull_request_target security split)
