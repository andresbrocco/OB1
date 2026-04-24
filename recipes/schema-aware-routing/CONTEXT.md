# CONTEXT.md — Schema-Aware Routing

## Purpose

Demonstrates a pattern for routing unstructured text input into multiple Supabase tables by using an LLM to extract structured metadata first. The extracted schema fields determine which tables receive writes — not every thought triggers every table.

## Responsibility Boundaries

- **Owns**: The full pipeline from raw text → LLM extraction → metadata-driven multi-table writes
- **Delegates to**: Caller-supplied LLM and embedding implementations (`extractMetadata`, `getEmbedding` are stubs)
- **Does not handle**: Authentication, MCP transport, Edge Function deployment, or queue/inbox management for pending person confirmations

## Key Concepts

**Schema-aware routing** — The LLM extraction prompt defines a schema whose fields are the sole inputs to routing logic. Each field maps to a downstream table or column; if a field is empty, the corresponding write is skipped.

**Three-pass person resolution** — Before creating a new `people` record, `findOrCreatePerson` runs three passes in order: (1) exact name/alias match with optional metadata backfill, (2) fuzzy first-name similarity (returns `"pending"` — requires human confirmation), (3) first-name collision detection (also returns `"pending"`). Only if all three passes fail does it create a new record.

**First-person intent gate** — Action items are written to `action_items` only when the extracted metadata indicates a first-person speaker commitment ("I need to", "I should", etc.). Third-party requests do not produce action item records. The `type` field is set to `"task"` by the same rule.

**`"pending"` person action** — A `PersonResult` with `action: "pending"` signals a fuzzy or collision match that was not auto-resolved. The person gets no `id`, so no `interactions` record is written for them. Production callers are expected to post a confirmation request to an external inbox or queue.

## Non-Obvious Details

- Embedding and LLM extraction run in parallel (`Promise.all`) before any database writes occur.
- The `thoughts` table is always written regardless of routing outcomes for other tables — raw capture is never dropped.
- The `interactions` write uses the full original text and embedding (not a summary), linking the raw thought to each resolved person.
- `action_items.linked_person_id` receives the first resolved person's ID, not all mentioned people — if multiple people are present, only the first one with an `id` is linked.
- Fuzzy matching is intentionally conservative: last-name-only overlap is explicitly excluded; minimum first-name length is 3 characters to avoid false matches on short names.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Two-stage review pipeline (deterministic gate + LLM qualitative review))
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Two-stage review pipeline)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Post-search filter extraction, Three-pass person resolution)
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Job contact vs professional contact, Pending person confirmation, Pipeline (application status lifecycle))
- **[extensions/professional-crm](../../extensions/professional-crm/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension bridge via denormalized note append, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Pending person confirmation, Trigger-managed last_contacted field)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Three-pass person resolution, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Pending person confirmation, Three-pass person resolution, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Three-pass person resolution, _enrichment_status)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Three-pass person resolution, Two-Prompt Extraction Sequence)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Content-type dispatch, Schema-aware routing)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Pending person confirmation, Three-pass person resolution)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, Noise filtering)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Dossier thought, Pending person confirmation)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, High-value categories, Per-category noise filtering)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), First-person intent gate)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Pending person confirmation, Three-pass person resolution)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, Gold-Found, Panning)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Selective LLM summarization)
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, Skip rules)
- **[recipes/thought-enrichment](../thought-enrichment/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Enrichment versioning, Pending person confirmation, Three-pass person resolution)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Hybrid filter+classify pipeline, Schema-aware routing)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Synthesizer catalogue)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Schema-aware routing, Synthesizer catalogue (plugin map))
- **[schemas](../../schemas/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Entity extraction queue with auto-trigger, Pending person confirmation, Three-pass person resolution)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Pending person confirmation, Thought-entity mention role and evidence, Three-pass person resolution)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Pending person confirmation, Three-pass person resolution)
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Diligence packet, Pending person confirmation)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Deterministic-first policy, Schema-aware routing)
- **[skills/n-agentic-harnesses](../../skills/n-agentic-harnesses/CONTEXT.md)** — Shares Multi-Provider LLM and Classification domain (Mode classification, Schema-aware routing)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Thread extraction, Three-pass person resolution)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, Signal diff vs digest)
