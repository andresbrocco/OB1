# CONTEXT.md — Work Operating Model Activation

## Purpose

A remote MCP server (Supabase Edge Function) that drives a structured, conversation-first interview to elicit a user's work operating model across five fixed layers and then renders that data into agent-ready export artifacts. The server manages session lifecycle, versioned layer checkpoints, and atomic entry storage, delegating all persistence to Supabase via two stored procedures.

## Responsibility Boundaries

- **Owns**: MCP tool surface, per-layer schema validation, entry normalization, export artifact rendering (USER.md, SOUL.md, HEARTBEAT.md, operating-model.json, schedule-recommendations.json), session/version resolution logic, and access-key authentication.
- **Delegates to**: Supabase stored procedures (`operating_model_start_session`, `operating_model_save_layer`) for transactional session and checkpoint writes; Supabase tables for all durable state.
- **Does not handle**: The AI agent interview conversation itself (that is the responsibility of the companion skill in `skills/work-operating-model`); user authentication beyond a shared `MCP_ACCESS_KEY`; multi-user routing (a single `DEFAULT_USER_ID` env var is used for all requests).

## Key Concepts

- **Five Layers**: The operating model is always structured across exactly five ordered layers: `operating_rhythms`, `recurring_decisions`, `dependencies`, `institutional_knowledge`, `friction`. The order is fixed and enforced at both the SQL and application levels.
- **Session Versioning**: Each elicitation run creates a new `profile_version` integer. A profile is `draft` while a session is active and transitions to `active` only when `generate_operating_model_exports` finalizes it. Querying without a version always resolves to the latest session, preferring in-progress over the committed profile version.
- **Checkpoint + Entry Separation**: Each approved layer produces one `operating_model_layer_checkpoints` row (a human-readable summary + raw payload) and N `operating_model_entries` rows (normalized, field-level structured data). The MCP server normalizes entries in TypeScript before passing them to the stored procedure; the SQL layer re-normalizes on insert.
- **source_confidence**: Distinguishes `confirmed` (explicit user statement) from `synthesized` (inferred from multiple examples). Used downstream by the AI agent to calibrate certainty.
- **Export Artifacts**: `generate_operating_model_exports` assembles five named artifacts from the approved checkpoints and entries. SOUL.md is an agent system-prompt fragment. HEARTBEAT.md is a triggered-checklist scaffold. USER.md is a human-readable layer summary. Both JSON exports use the same grouped entry data.
- **Layer Detail Validators**: Each of the five layers has a Zod sub-schema enforced on the `details` JSONB field (e.g., `operating_rhythms` requires `time_windows` and `energy_pattern`; `friction` requires `frequency`, `time_cost`, `current_workaround`). Entries cannot be saved with a mismatched detail shape.

## Non-Obvious Details

- The `resolveVersion` function has a deliberate priority order: explicit version request > latest in-progress session > committed profile version. This means a session that was started but never completed will shadow the last completed profile version in query results.
- `generate_operating_model_exports` enforces that all five layers have approved checkpoints before it will produce output. Partial exports are not possible.
- The session finalization in `generate_operating_model_exports` writes to both `operating_model_sessions` (status → completed) and `operating_model_profiles` (current_version bumped, status → active) in two separate Supabase update calls without a wrapping transaction. A failure between the two would leave the session completed but the profile still on the previous version.
- The `@ts-ignore` on line 843 suppresses a Deno-specific stream compatibility issue with the `duplex: "half"` property; this is a known Deno/Hono limitation, not an application logic issue.
- Access key authentication accepts the key from three sources in priority order: query param `key`, header `x-brain-key`, header `x-access-key`.

## Related Modules

- **[dashboards/open-brain-dashboard-next](../../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (quality audit (quality_score <= 29), source_confidence)
- **[dashboards/open-brain-dashboard-next/app/api](../../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Audit quality threshold, source_confidence)
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Interview stages, Layer Detail Validators)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Importance scale (0-6, 6 is user-only), source_confidence)
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Confidence gating, Per-type thresholds, source_confidence)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Provides start_operating_model_session (MCP tool), save_operating_model_layer (MCP tool), query_operating_model (MCP tool), ... consumed by this module
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session Versioning, Session splitting)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Quality Gate, source_confidence)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Body cleaning pipeline, Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Checkpoint + Entry Separation, Cursor-based resumability)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Session Versioning, Transcript assembly)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (--redo flag, Checkpoint + Entry Separation)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), Meta latin1/UTF-8 encoding repair)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Checkpoint + Entry Separation, Dynamic loop rescheduling)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Topic shift detection)
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Speaker Consolidation, Thread)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Two-sheet import (Conversations + Memory))
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (LLM classification prompt with importance/confidence calibration, source_confidence)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Checkpoint + Entry Separation, Phase toggles)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Thread eligibility gate)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Import and Export Pipelines domain (Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), Tweet batching, Twitter JS export format)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (importance/quality_score ranking signals, source_confidence)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Async queue with content-addressed re-queue, Checkpoint + Entry Separation)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Extraction threshold and quality gates, source_confidence)
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Conviction state, Decision-readiness, Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Layer Detail Validators)
- **[skills/financial-model-review](../../skills/financial-model-review/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Layer Detail Validators, Structural risk vs. business risk)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Converter preference (auto/native/markitdown), Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (ConversionResult, Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md), export bundles)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Speaker Consolidation)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Layer Detail Validators, Structural questions framework)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Depends on for Structured AI skill for interviewing users about how their work actually runs, saving approved results into Open Brain via paired recipe MCP tools, and generating agent-ready operating model exports
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Quality Scoring and Confidence domain (Five-principle evaluation, source_confidence)
