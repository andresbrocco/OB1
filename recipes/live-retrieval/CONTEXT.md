# CONTEXT.md — Live Retrieval

## Purpose

Defines an ambient, proactive AI skill that automatically surfaces relevant Open Brain thoughts during active work sessions. It operates as the "read side of the flywheel" — triggering semantic searches in response to detected topic shifts rather than explicit user commands.

## Responsibility Boundaries

- **Owns**: Topic-shift detection logic, hit/miss surface rules, session-scoped deduplication, retrieval logging format, and failure behavior policy
- **Delegates to**: The `search_thoughts` and `list_thoughts` MCP tools for actual data retrieval
- **Does not handle**: Pre-meeting briefings (Life Engine), brainstorm context loading (Panning for Gold), or direct full-text search on user command

## Key Concepts

- **Flywheel (read side)**: Live Retrieval is explicitly framed as the consumption half of the capture-then-retrieve loop; it only reads, never writes
- **Topic shift**: A named entity (person, project, technology, concept) appearing in the user's message that was absent from the prior 3 messages — the primary trigger condition
- **Hit threshold**: Score > 0.6 from `search_thoughts` determines whether results are surfaced; below this they are treated as misses
- **Session cap**: A hard limit of 3 retrievals per session prevents over-triggering; exceeding it before the session midpoint signals overly sensitive detection
- **Silent-on-miss contract**: Failed or empty searches must never be acknowledged to the user — silence is the specified failure mode

## Non-Obvious Details

- The skill instructs the AI to log every search (hit or miss) to `.claude/live-retrieval-log.jsonl` in the project root, enabling hit-rate analysis after 10 sessions
- The log review thresholds (hit rate < 20% → broaden detection, > 80% → narrow triggers, avg score < 0.5 → raise threshold) are prescriptive tuning guidelines baked into the skill definition
- Deduplication is session-scoped by thought ID — the same thought must never be shown twice in one session
- The skill fires at session start unconditionally (pulls 3 recent thoughts + 5 latest), in addition to topic-shift detection mid-session
- MCP unavailability, errors, and timeouts all silently no-op; the user is never informed of infrastructure failures

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comment via ob1-automated-review marker, Session-scoped deduplication)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent PR comments, Session-scoped deduplication)
- **[dashboards/open-brain-dashboard/src/lib](../../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Silent-on-miss contract, ephemeral IDs)
- **[extensions/family-calendar](../../extensions/family-calendar/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (NULL family_member_id for household-wide events, Silent-on-miss contract)
- **[extensions/home-maintenance](../../extensions/home-maintenance/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Silent-on-miss contract, frequency_days=NULL for one-time tasks)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Session-scoped deduplication, re-extraction idempotency)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), pgvector cosine distance via raw SQL)
- **[integrations/kubernetes-deployment/k8s](../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), match_thoughts RPC equivalent)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation tree / branch resolution, Session splitting, Topic shift detection)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Session-scoped deduplication, Sync log, Two-layer dedup)
- **[recipes/entity-wiki](../entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Semantic expansion, Session cap (max 3 retrievals))
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Duplicate row, Session-scoped deduplication)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Day-hash dedup via sync log, Session-scoped deduplication)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Conversation normalization, Topic shift detection, Transcript assembly)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Session-scoped deduplication)
- **[recipes/journals-blogger-import](../journals-blogger-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Session-scoped deduplication)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Session-scoped deduplication)
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Hit threshold (score > 0.6), Local embedding via Ollama, Retrieval log, Session cap (max 3 retrievals))
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Dual deduplication (sync log + content fingerprint), Session-scoped deduplication)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Thread, Topic shift detection)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Topic shift detection, Two-sheet import (Conversations + Memory))
- **[recipes/repo-learning-coach](../repo-learning-coach/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Flywheel (read side), Understanding State)
- **[recipes/repo-learning-coach/server](../repo-learning-coach/server/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Flywheel (read side), UnderstandingState)
- **[recipes/repo-learning-coach/src](../repo-learning-coach/src/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), RelatedThoughts, Retrieval log, Session cap (max 3 retrievals))
- **[recipes/repo-learning-coach/src/lib](../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), RelatedThoughtSummary, Retrieval log, Session cap (max 3 retrievals))
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Idempotent upsert via thought_edges_upsert RPC, Session-scoped deduplication)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), match_thoughts)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility (content-weight gating), Topic shift detection)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Thread eligibility gate, Topic shift detection)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Session Versioning, Topic shift detection)
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprinting, Session-scoped deduplication)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Session-scoped deduplication, idempotent schema migration)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Null Semantics and Upsert Conflict Resolution domain (Silent-on-miss contract, Temporal validity with NULL semantics, Upsert NULL-wins conflict resolution)
- **[server](../../server/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Session cap (max 3 retrievals), match_thoughts RPC (pgvector similarity search))
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Open Brain deduplication workflow, Session-scoped deduplication)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Conversation and Thread Processing domain (Speaker Consolidation, Topic shift detection)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Live search upgrade, Retrieval log, Session cap (max 3 retrievals))
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Flywheel (read side), Lean memory discipline)
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Flywheel (read side), OB1-connected mode vs. direct-chat mode, World-model paradigm (vector database / structured ontology / signal-fidelity))
