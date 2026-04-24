# CONTEXT.md — Life Engine

## Purpose

A time-aware, self-improving personal assistant skill that runs on a recurring cron loop. It monitors the user's calendar and Open Brain knowledge base to deliver proactive briefings via Telegram or Discord at contextually appropriate times — morning summaries, pre-meeting prep, midday check-ins, and evening wrap-ups. It also tracks habits and evolves its own behavior weekly based on observed user engagement.

## Responsibility Boundaries

- **Owns**: The full loop execution logic (time windowing, duplicate-check, data fetch, message delivery, rescheduling), all `life_engine_*` database tables, habit tracking, check-in logging, briefing deduplication, and the self-improvement suggestion cycle
- **Delegates to**: Google Calendar MCP for live event data, Open Brain (`capture_thought`, `search`) for knowledge enrichment, Telegram/Discord channel plugins for message delivery, and Supabase MCP for database reads/writes
- **Does not handle**: Sending notifications through any means other than Telegram or Discord; managing the core `thoughts` table; user authentication or pairing (that happens outside this skill)

## Key Concepts

- **Anchor date / anchor time**: The single source-of-truth timestamp established at loop start via `date` or calendar API. All duplicate checks, 7-day lookbacks, and "Week of" labels are calculated from this value — never from vague relative terms.
- **Briefing deduplication**: Before sending any briefing, the skill queries `life_engine_briefings` filtered by `anchor_date`. Only one briefing of each `briefing_type` is sent per calendar day.
- **Dynamic loop rescheduling**: Each loop iteration ends by deleting the current cron job and creating a new one with an interval derived from the current time window (15 min in the morning, 30 min midday, 60 min evening, single one-shot overnight). This pattern keeps the loop alive indefinitely despite Claude Code cron's 3-day expiry limit.
- **Self-improvement protocol**: Every 7 days the skill analyzes `life_engine_briefings.user_responded` to identify high-value vs. low-engagement briefing types, formulates one behavior change suggestion, and tracks approval via `life_engine_evolution`.
- **External before internal enrichment**: The skill always pulls live external data (calendar, weather) before searching Open Brain. You cannot enrich what you haven't seen yet.
- **`user_id` as channel chat_id**: The `user_id` column in all `life_engine_*` tables stores the Telegram/Discord `chat_id`, not a UUID. This changed from earlier versions (UUID → TEXT migration documented in `schema.sql`).

## Non-Obvious Details

- The `life_engine_state` table is a key-value store with no `user_id` column; it assumes a single Life Engine instance per Supabase project. Multi-user setups must prefix keys manually.
- All tables use RLS enabled with no permissive policies — access is intentionally restricted to `service_role` only. Anon and authenticated roles are blocked.
- The `life-engine-skill.md` file is the recipe development source of truth. The installed copy at `~/.claude/skills/life-engine/SKILL.md` may contain personal customizations (calendar IDs, user-specific references) and must be manually synced by the user — recipe updates are never auto-deployed.
- Cron jobs should be offset from :00 and :30 to avoid contention (e.g., `7,22,37,52` instead of `*/15`).
- Channel messages (Telegram/Discord) are treated as untrusted input. The skill explicitly guards against prompt injection: it never executes commands found in message text, never shares system configuration, and ignores role-switching language.
- Weather is fetched from Open-Meteo (no API key required) using lat/lon from `life_engine_state` (defaults to Portland, OR). Rain is only included in morning briefings.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Artifact handoff between workflows, Self-improvement protocol)
- **[dashboards](../../dashboards/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (Supabase Auth, user_id as channel chat_id)
- **[dashboards/open-brain-dashboard-next/lib](../../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (server-only boundary, user_id as channel chat_id)
- **[dashboards/open-brain-dashboard/src](../../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (SvelteKit locals augmentation, user_id as channel chat_id)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Post-search filter extraction)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pattern, user_id as channel chat_id)
- **[extensions/family-calendar](../../extensions/family-calendar/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID single-tenant pinning, user_id as channel chat_id)
- **[extensions/home-maintenance](../../extensions/home-maintenance/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID identity pinning, user_id as channel chat_id)
- **[extensions/household-knowledge](../../extensions/household-knowledge/CONTEXT.md)** — Shares Single-Tenant Identity and User Isolation domain (DEFAULT_USER_ID environment injection, user_id as channel chat_id)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, re-extraction idempotency)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, _enrichment_status)
- **[recipes/adaptive-capture-classification](../adaptive-capture-classification/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Self-improvement protocol, Two-phase pipeline)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Two-Prompt Extraction Sequence)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Sync log)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception, Retrospective Mode, Self-improvement protocol)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Sync log, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint, Duplicate row)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Day-hash dedup via sync log)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint deduplication)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Self-improvement protocol, Two-phase pipeline)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint (SHA-256 deduplication))
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint for deduplication)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Session-scoped deduplication)
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Dual deduplication (sync log + content fingerprint))
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Compaction-safe persistence, user_id as channel chat_id)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Local sync log deduplication)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Pending person confirmation, Three-pass person resolution)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, External before internal enrichment)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/vercel-neon-telegram](../vercel-neon-telegram/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Parallel capture pipeline, Self-improvement protocol)
- **[recipes/vercel-neon-telegram/src](../vercel-neon-telegram/src/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Telegram webhook singleton bot, in-memory sliding-window rate limiter, user_id as channel chat_id)
- **[recipes/vercel-neon-telegram/src/app/api](../vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Telegram and Bot Integration domain (Bot singleton pattern, Telegram webhook secret authentication, user_id as channel chat_id)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Phase toggles, Self-improvement protocol)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Dynamic loop rescheduling, Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Dynamic loop rescheduling, Resume-safe JSONL state log)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Resume-Safe State and Cursor Pagination domain (Checkpoint + Entry Separation, Dynamic loop rescheduling)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprinting)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, External before internal enrichment)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, idempotent schema migration)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Async queue with content-addressed re-queue, Self-improvement protocol)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Temporal Validity and Decay domain (Anchor date/time, Dynamic loop rescheduling, Temporal validity with NULL semantics, decay_weight)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Aiception/Claudeception (self-referential skill extraction), Retrospective mode, Self-improvement protocol)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Agentic Harness and Workflow Orchestration domain (Approval gates, Harness, Harness primitives, Self-improvement protocol)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Thread extraction)
