# CONTEXT.md — World Model Diagnostic

## Purpose

A structured conversational skill that guides an AI agent through a 20-minute diagnostic session to assess a company's readiness to build a persistent world model. The skill produces a paradigm classification, a boundary-layer audit, and a prioritized build sequence — not a readiness score.

## Responsibility Boundaries

- **Owns**: The diagnostic protocol, paradigm-mapping rules, five-principle evaluation criteria, boundary-layer audit structure, and the persistence contract for saving session artifacts to Open Brain
- **Delegates to**: The AI client for conversational execution; Open Brain search/capture tools (when available) for context retrieval and artifact persistence
- **Does not handle**: Actual data retrieval, schema design, or implementation — it only produces a strategic starting sequence

## Key Concepts

- **World-model paradigm**: One of three architectural fits — `vector database`, `structured ontology`, or `signal-fidelity` — assigned based on company size, regulatory context, and signal quality
- **Boundary layer**: The explicit line separating factual information routing from editorial interpretation; the central diagnostic object. Most companies lack this boundary, which the skill is designed to surface
- **Simulated judgment**: Situations where automation appears to have replaced human interpretation but has not — the routing still conceals embedded judgment calls
- **Act on this / Interpret this first**: The two labels used during the boundary audit to classify each information flow
- **OB1-connected mode vs. direct-chat mode**: The skill runs identically in both modes; when Open Brain tools are present, it persists three structured artifacts (`intake`, `boundary-audit`, `assessment`) tagged with `[world-model-diagnostic/...]` prefixes
- **Five-principle evaluation**: A qualitative readout across signal fidelity, earned structure, outcome encoding, organizational resistance, and time in system — all classified by named tiers, never by numeric scores

## Non-Obvious Details

- The skill explicitly prohibits numeric readiness scores; every conclusion must be labeled as `Firm finding`, `Inference`, or `Open question`. This is a design constraint enforced in the non-negotiable rules, not a stylistic preference.
- Paradigm mapping uses a priority tiebreaker sequence (highest-fidelity signal > cost of bad interpretation > available senior judgment) when company cues conflict.
- The knowledge-work case (conversations, docs, soft context) is identified as the hardest and most common company type; it still maps to `vector database` but requires aggressive boundary-layer work before any retrieval layer is built.
- The persistence contract saves exactly three lean artifacts unless the user opts out — individual question answers are explicitly not persisted.
- Tool name resolution is intentionally fuzzy: the skill looks for tools by behavior, not by exact name, to handle namespaced deployments.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Simulated judgment)
- **[.github](../../.github/CONTEXT.md)** — Shares Prompt Injection and Security domain (Simulated judgment, security-blocked and needs-maintainer-triage label lifecycle)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Prompt Injection and Security domain (Simulated judgment, pull_request vs pull_request_target security split)
- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, quality audit (quality_score <= 29))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Five-principle evaluation)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Importance scale (0-6, 6 is user-only))
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Five-principle evaluation, Per-type thresholds)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Source Confidence)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Quality Gate)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Simulated judgment)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Flywheel (read side), OB1-connected mode vs. direct-chat mode, World-model paradigm (vector database / structured ontology / signal-fidelity))
- **[recipes/obsidian-vault-import](../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Prompt Injection and Security domain (Secret scanning, Simulated judgment)
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Flywheel closure, OB1-connected mode vs. direct-chat mode, World-model paradigm (vector database / structured ontology / signal-fidelity))
- **[recipes/repo-learning-coach](../../recipes/repo-learning-coach/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, OB1-connected mode vs. direct-chat mode, Understanding State, World-model paradigm (vector database / structured ontology / signal-fidelity))
- **[recipes/repo-learning-coach/server](../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, OB1-connected mode vs. direct-chat mode, UnderstandingState, World-model paradigm (vector database / structured ontology / signal-fidelity))
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, OB1-connected mode vs. direct-chat mode, UnderstandingState, World-model paradigm (vector database / structured ontology / signal-fidelity))
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridgeState, OB1-connected mode vs. direct-chat mode, UnderstandingState, World-model paradigm (vector database / structured ontology / signal-fidelity))
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Firm finding / Inference / Open question labeling, Five-principle evaluation, Investor path)
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, LLM classification prompt with importance/confidence calibration)
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Simulated judgment)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, source_confidence)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, importance/quality_score ranking signals)
- **[skills/claudeception](../claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, Five-principle evaluation)
- **[skills/deal-memo-drafting](../deal-memo-drafting/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Deal memo, Diligence packet, Firm finding / Inference / Open question labeling, Five-principle evaluation, IC memo)
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Fatal issues vs. caution flags vs. acceptable simplifications, Firm finding / Inference / Open question labeling, Five-principle evaluation, Investor-grade / Board-grade / Internal planning standard, Structural risk vs. business risk)
- **[skills/heavy-file-ingestion](../heavy-file-ingestion/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, Quality flags)
- **[skills/heavy-file-ingestion/scripts](../heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, quality flags)
- **[skills/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Prompt Injection and Security domain (Mary's Law Check, Simulated judgment)
- **[skills/weekly-signal-diff](../weekly-signal-diff/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (OB1-connected mode vs. direct-chat mode, Personalization vs discovery balance, World-model paradigm (vector database / structured ontology / signal-fidelity))
- **[skills/work-operating-model](../work-operating-model/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Firm finding / Inference / Open question labeling, Five-principle evaluation)
