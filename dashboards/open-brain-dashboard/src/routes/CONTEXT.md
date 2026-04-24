# CONTEXT.md — Routes

## Purpose

Defines the top-level SvelteKit route tree for the Open Brain dashboard. Provides the authenticated shell layout and the single-page search/capture interface for browsing and adding thoughts.

## Responsibility Boundaries

- **Owns**: Root layout (auth guard, global shell chrome), root page (search UI, capture form, thought display)
- **Delegates to**: `$lib/api` for all data fetching and mutation; `$lib/types` for thought type definitions
- **Does not handle**: Sub-routes (signin, signout) — those live in sibling directories not present here

## Key Concepts

- **`+layout.server.ts` auth guard**: Server-side `load` function redirects any unauthenticated request (where `locals.user` is absent) to `/signin`, except for the `/signin` path itself. `locals.user` is populated upstream in a hooks file (not in this directory).
- **`latestResultKey`**: A composite key (`created_at::content`) that identifies the most recent search result. The top card in a new search batch is highlighted with a pulsing ring to indicate it is the newest match.
- **Incremental result merging**: `loadThoughts` merges new server results with the existing `thoughts` array using the composite key as a dedup identity. This means repeated searches accumulate results in-memory rather than replacing them, preserving previously found thoughts.

## Non-Obvious Details

- Clearing search query (`clearSearchQuery`) does not clear results — the comment in `handleSearchInput` is intentional. Results persist until the user explicitly clicks "Clear results".
- Stats (`loadStats`) pull topic and person aggregates from the API, but after a search, `extractFilters` overwrites those with values derived only from current in-memory results. The displayed filter chips therefore reflect the current result set, not the global corpus.
- `thoughtKey` uses `created_at + content` rather than an `id` field, implying the `Thought` type may not always guarantee a stable unique `id` in this context.

## Related Modules

- **[dashboards](../../../CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../../CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, SSR auth)
- **[dashboards/open-brain-dashboard-next](../../../open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../../../open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Restricted content unlock, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/components](../../../open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Restricted content passphrase gating)
- **[dashboards/open-brain-dashboard-next/lib](../../../open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Auth guard via layout.server.ts, SvelteKit locals augmentation, latestResultKey highlight)
- **[docs](../../../../docs/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Query-parameter auth pattern)
- **[extensions/household-knowledge](../../../../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, MCP_ACCESS_KEY pre-shared key authentication)
- **[extensions/meal-planning](../../../../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Household member RLS via JWT role claim)
- **[integrations](../../../../integrations/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, entity_extraction_queue)
- **[integrations/entity-extraction-worker](../../../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (ExtractionCostCapError, Post-search filter extraction, entity_extraction_queue)
- **[integrations/entity-extraction-worker/_shared](../../../../integrations/entity-extraction-worker/_shared/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, _enrichment_status)
- **[integrations/kubernetes-deployment](../../../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, MCP_ACCESS_KEY authentication)
- **[recipes/bring-your-own-context](../../../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, Two-Prompt Extraction Sequence)
- **[recipes/chatgpt-conversation-import](../../../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Pyramid summaries)
- **[recipes/claudeception](../../../../recipes/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction, Post-search filter extraction)
- **[recipes/entity-wiki](../../../../recipes/entity-wiki/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Slug collision resolution)
- **[recipes/infographic-generator](../../../../recipes/infographic-generator/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Manifest file)
- **[recipes/life-engine](../../../../recipes/life-engine/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (External before internal enrichment, Post-search filter extraction)
- **[recipes/obsidian-vault-import](../../../../recipes/obsidian-vault-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Two-tier chunking (heading split + LLM fallback))
- **[recipes/schema-aware-routing](../../../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Pending person confirmation, Post-search filter extraction, Three-pass person resolution)
- **[recipes/thought-enrichment](../../../../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Sensitivity tiers (standard/personal/restricted))
- **[recipes/vercel-neon-telegram/src](../../../../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../../../../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Bearer token authentication, Telegram webhook secret authentication)
- **[recipes/wiki-compiler](../../../../recipes/wiki-compiler/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Compile manifest, Compiled wiki, Incremental result merging)
- **[recipes/wiki-synthesis](../../../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, wiki vs entity-wiki distinction)
- **[schemas](../../../../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../../../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, sensitivity_tier)
- **[schemas/entity-extraction](../../../../schemas/entity-extraction/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Canonical entity / normalized name deduplication, Post-search filter extraction, Thought-entity mention role and evidence)
- **[server](../../../../server/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, x-brain-key access key auth)
- **[skills](../../../../skills/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Auth guard via layout.server.ts, Variants, latestResultKey highlight)
- **[skills/claudeception](../../../../skills/claudeception/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Extraction threshold and quality gates, Post-search filter extraction)
- **[skills/heavy-file-ingestion](../../../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Dashboard and Frontend Patterns domain (Auth guard via layout.server.ts, Client variants and build exports, latestResultKey highlight)
- **[skills/heavy-file-ingestion/scripts](../../../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (.ob1 output directory, Incremental result merging)
- **[skills/panning-for-gold](../../../../skills/panning-for-gold/CONTEXT.md)** — Shares Entity Extraction and Enrichment domain (Post-search filter extraction, Thread extraction)
