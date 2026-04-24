# CONTEXT.md — Deal Memo Drafting

## Purpose

Provides a structured AI skill for drafting investor-grade deal memos, IC memos, partnership memos, and acquisition briefs from pre-existing diligence materials. The skill does not perform diligence itself — it converts evidence already gathered (research findings, model review outputs, meeting notes) into a decision-ready recommendation document while explicitly surfacing gaps and open questions.

## Responsibility Boundaries

- **Owns**: Memo framing, evidence inventory and categorization, structured drafting of thesis/market/economics/risk/recommendation sections, and explicit labeling of inference vs. confirmed facts
- **Delegates to**: `competitive-analysis` for market mapping, `financial-model-review` for economics, `research-synthesis` for source-backed findings, `meeting-synthesis` for management/partner conversation extraction
- **Does not handle**: Raw diligence generation, standalone market mapping without a memo deliverable, transcript cleanup, or financial model review in isolation

## Key Concepts

- **IC memo**: Investment Committee memo — formal document supporting a fund or committee-level investment decision
- **Deal memo**: Broader term covering investment, partnership, acquisition, or strategic decision documents
- **Diligence packet**: The upstream evidence set (research notes, model outputs, meeting summaries) that the skill drafts from
- **Conviction state**: The current degree of confidence in the thesis, which the skill requires as explicit input rather than inferring
- **Decision-readiness**: The quality bar the memo targets — ends with a clear recommendation, explicit confidence level, and enumerated next-step requirements

## Non-Obvious Details

- The skill enforces epistemic honesty as a core constraint: it will explicitly label weak sections as weak when the underlying diligence is incomplete, rather than filling gaps with confident prose. This is intentional and documented as a rule, not a deficiency.
- A valid output of this skill is a memo that recommends "not ready" or "do not proceed yet" — the skill is designed to support negative or conditional recommendations, not just affirmative ones.
- Open Brain integration is optional: search can surface prior deal notes or thesis fragments, and capture can store the final memo or recommendation summary, but neither is required for the skill to function.

## Related Modules

- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Diligence packet, Job contact vs professional contact, Pipeline (application status lifecycle))
- **[extensions/professional-crm](../../extensions/professional-crm/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension bridge via denormalized note append, Diligence packet, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Trigger-managed last_contacted field)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Conviction state, Decision-readiness, Operating Model Layers)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Diligence packet, Dossier thought)
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (ACT NOW / RESEARCH MORE / PARK / KILL, Conviction state, Decision-readiness)
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Provides SKILL.md prompt/skill definition consumed by this module
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Diligence packet, Pending person confirmation)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Conviction state, Decision-readiness, Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Layer Detail Validators)
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Depends on for Investor-first AI skill pack for reviewing existing financial models, forecasts, and scenario sets for assumption quality, structural risk, and decision usefulness
- **[skills/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (COS items, Conviction state, Decision-readiness, Verdict taxonomy (ACT NOW / RESEARCH MORE / PARK IT / KILL IT))
- **[skills/weekly-signal-diff](../weekly-signal-diff/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Conviction state, Decision-readiness, Structural questions framework)
- **[skills/work-operating-model](../work-operating-model/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Contradiction pass, Deal memo, Diligence packet, IC memo)
- **[skills/world-model-diagnostic](../world-model-diagnostic/CONTEXT.md)** — Shares Investment and Deal Analysis domain (Deal memo, Diligence packet, Firm finding / Inference / Open question labeling, Five-principle evaluation, IC memo)
