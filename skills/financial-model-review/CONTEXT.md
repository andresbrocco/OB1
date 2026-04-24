# CONTEXT.md — Financial Model Review

## Purpose

Provides an investor-first AI skill pack for reviewing an existing financial model, forecast, or sensitivity analysis. The goal is to assess whether a model's assumptions, structure, scenarios, and logic are sound enough to support a real decision — not to build or extend the model itself.

## Responsibility Boundaries

- **Owns**: Assumption quality review, structural and logic risk identification, scenario adequacy assessment, and converting model findings into a decision-ready verdict
- **Delegates to**: `deal-memo-drafting` for final memo writing when the model review is one input among many; `competitive-analysis` for market benchmark validation of assumptions; `research-synthesis` for source-backed contradiction handling
- **Does not handle**: Building models from scratch, market or competitor research without a model artifact, or general document synthesis

## Key Concepts

- **Investor-grade / Board-grade / Internal planning standard**: Three distinct review bars that determine how rigorously assumptions and scenarios must be defended
- **Model shape**: The categorization pass — revenue model, cost structure, cash runway, valuation, scenario design, and outputs — before diving into assumptions
- **Fatal issues vs. caution flags vs. acceptable simplifications**: The three-tier verdict classification that separates blocking problems from advisory notes
- **Structural risk vs. business risk**: A distinction the skill is designed to maintain — structural issues (missing drivers, circular logic, hard-codes) are separate from business-level concerns (market size, competition)
- **Roll-forwards**: Period-to-period balance sheet or metric carry-forward formulas that are a common source of hidden logic errors in spreadsheet models

## Non-Obvious Details

- The skill is explicitly scoped to reviewing what exists. It must not drift into model building mid-session. The SKILL.md notes this directly as a risk to guard against.
- The skill distinguishes between what is visible (formulas, exported assumptions) and what cannot be verified (underlying spreadsheet logic not shared). Reviewers are instructed to declare limits of visibility rather than infer correctness.
- Open Brain integration is optional: the skill can search for prior model versions, management claims, or historical notes, and can capture the review memo after completion — but neither is required for the core review workflow.
- The audience split (investors/diligence teams as primary; operators as secondary) affects framing. The default framing is investor-first, which means assumptions are held to an external scrutiny standard rather than internal planning tolerance.

## Related Modules

- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, quality audit (quality_score <= 29))
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, Fatal issues vs. caution flags vs. acceptable simplifications)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, details JSONB freeform metadata field)
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Interview stages, Structural risk vs. business risk)
- **[extensions/meal-planning](../../extensions/meal-planning/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSONB ingredient and shopping item storage, Model shape)
- **[integrations](../../integrations/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, Shared config with sensitivity tiers)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, Importance scale (0-6, 6 is user-only))
- **[recipes/adaptive-capture-classification](../../recipes/adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Fatal issues vs. caution flags vs. acceptable simplifications, Per-type thresholds)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Operating Model Layers, Structural risk vs. business risk)
- **[recipes/claudeception](../../recipes/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, Quality Gate)
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Structural risk vs. business risk)
- **[recipes/perplexity-conversation-import](../../recipes/perplexity-conversation-import/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (JSON profile rows, Model shape)
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Consumed by for Composable orchestration scaffold that sequences five OB1 skills into operator or investor decision pipelines
- **[recipes/thought-enrichment](../../recipes/thought-enrichment/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, Type backfill (metadata.type promotion))
- **[recipes/vercel-neon-telegram/src](../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, ThoughtMetadata)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, ThoughtMetadata)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Layer Detail Validators, Structural risk vs. business risk)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, importance/quality_score ranking signals)
- **[skills/claudeception](../claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, Fatal issues vs. caution flags vs. acceptable simplifications)
- **[skills/deal-memo-drafting](../deal-memo-drafting/CONTEXT.md)** — Consumed by for AI skill for drafting structured deal, IC, partnership, or acquisition memos from pre-existing diligence materials
- **[skills/heavy-file-ingestion](../heavy-file-ingestion/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, Quality flags)
- **[skills/heavy-file-ingestion/scripts](../heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Fatal issues vs. caution flags vs. acceptable simplifications, quality flags)
- **[skills/n-agentic-harnesses](../n-agentic-harnesses/CONTEXT.md)** — Shares JSONB and Schema Metadata domain (Model shape, Product shape)
- **[skills/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Structural risk vs. business risk, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../weekly-signal-diff/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Structural questions framework, Structural risk vs. business risk)
- **[skills/work-operating-model](../work-operating-model/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Fatal issues vs. caution flags vs. acceptable simplifications, Investor-grade / Board-grade / Internal planning standard, Structural risk vs. business risk)
- **[skills/world-model-diagnostic](../world-model-diagnostic/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Fatal issues vs. caution flags vs. acceptable simplifications, Firm finding / Inference / Open question labeling, Five-principle evaluation, Investor-grade / Board-grade / Internal planning standard, Structural risk vs. business risk)
