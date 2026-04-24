# CONTEXT.md — Weekly Signal Diff

## Purpose

Provides a reusable AI skill for transforming a week's worth of market or industry news into a personalized set of structural changes ("diffs"), rather than a headline roundup. The skill queries Open Brain memory to weight the output toward the user's actual projects and interests, and optionally uses live web search for fresh, cited evidence.

## Responsibility Boundaries

- **Owns**: The procedural prompt logic for running a weekly structural diff, output format, scoring criteria, and capture behavior
- **Delegates to**: The AI client's Open Brain tools for memory retrieval and capture; OpenRouter/Perplexity Sonar (optional) for live web retrieval
- **Does not handle**: Scheduling or automation triggering, database writes directly, or news ingestion pipelines

## Key Concepts

- **Signal diff vs digest**: The core distinction — a digest reports what happened; a diff identifies what structurally changed (shifted constraints, leverage, pricing assumptions, distribution). The skill explicitly filters for the latter.
- **Starter universe**: A bootstrap layer of 10 suggested categories and 30 suggested companies defined in `references/starter-universe.md`. Used only when the user has no watchlist. It prevents blank-page syndrome but is not authoritative.
- **Structural questions**: A fixed set of interrogative lenses applied to each candidate signal (who gained leverage, what got cheaper, what dependency was exposed, etc.) to separate noise from structural change.
- **Personalization vs discovery balance**: Open Brain memory re-ranks the watchlist but must not collapse it to only already-known entities. The skill preserves a baseline discovery margin intentionally.
- **Live search upgrade**: An optional retrieval mode documented in `references/live-search-upgrade.md`. When OpenRouter is available, the skill prefers Perplexity Sonar models for the retrieval pass and defers synthesis to the local client.

## Non-Obvious Details

- The skill is portable across AI clients (Claude Code, Codex, Cursor). It avoids assuming fixed tool names for Open Brain operations, instead instructing the agent to use whatever search/capture tools are available in the environment.
- The output is intended to be saved back into Open Brain as a durable weekly digest so future runs can compare against prior diffs. Week-over-week comparability depends on consistent output structure, which the format section enforces.
- A good diff contains only 3–7 structural shifts. More than that typically means the scoring step was skipped or noise was not filtered.
- If evidence is thin for a given week, the skill instructs the agent to say so explicitly rather than pad the output.

## Related Modules

- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Interview stages, Structural questions framework)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Kubernetes self-hosted MCP server, Starter universe bootstrap)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Starter universe bootstrap, Supabase replacement pattern)
- **[integrations/kubernetes-deployment/k8s](../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Co-located pod pattern, ConfigMap-embedded SQL, Starter universe bootstrap, Supabase schema parity, hostPath volume)
- **[recipes/bring-your-own-context](../../recipes/bring-your-own-context/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Operating Model Layers, Structural questions framework)
- **[recipes/chatgpt-conversation-import](../../recipes/chatgpt-conversation-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Signal-based filtering)
- **[recipes/email-history-import](../../recipes/email-history-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Noise filtering, Signal diff vs digest)
- **[recipes/entity-wiki](../../recipes/entity-wiki/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, Semantic expansion)
- **[recipes/google-activity-import](../../recipes/google-activity-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (High-value categories, Per-category noise filtering, Signal diff vs digest)
- **[recipes/journals-blogger-import](../../recipes/journals-blogger-import/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Signal diff vs digest)
- **[recipes/live-retrieval](../../recipes/live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Live search upgrade, Retrieval log, Session cap (max 3 retrievals))
- **[recipes/local-ollama-embeddings](../../recipes/local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Live search upgrade, Local embedding via Ollama)
- **[recipes/panning-for-gold](../../recipes/panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Gold-Found, Panning, Signal diff vs digest)
- **[recipes/repo-learning-coach](../../recipes/repo-learning-coach/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Personalization vs discovery balance, Understanding State)
- **[recipes/repo-learning-coach/server](../../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Personalization vs discovery balance, UnderstandingState)
- **[recipes/repo-learning-coach/src](../../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Bootstrap, Starter universe bootstrap)
- **[recipes/repo-learning-coach/src/lib](../../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (BootstrapData, Starter universe bootstrap)
- **[recipes/research-to-decision-workflow](../../recipes/research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Skip rules)
- **[recipes/schema-aware-routing](../../recipes/schema-aware-routing/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (First-person intent gate, Signal diff vs digest)
- **[recipes/vercel-neon-telegram/src/lib](../../recipes/vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, match_thoughts)
- **[recipes/wiki-synthesis](../../recipes/wiki-synthesis/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../../recipes/wiki-synthesis/scripts/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Signal diff vs digest, Thread eligibility gate)
- **[recipes/work-operating-model-activation](../../recipes/work-operating-model-activation/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five Layers (operating_rhythms, recurring_decisions, dependencies, institutional_knowledge, friction), Layer Detail Validators, Structural questions framework)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, Two-phase full-text search (GIN tsvector + ILIKE fallback))
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Kubernetes and Infrastructure Deployment domain (Starter universe bootstrap, idempotent schema migration)
- **[server](../../server/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, match_thoughts RPC (pgvector similarity search))
- **[skills/deal-memo-drafting](../deal-memo-drafting/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Conviction state, Decision-readiness, Structural questions framework)
- **[skills/financial-model-review](../financial-model-review/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Structural questions framework, Structural risk vs. business risk)
- **[skills/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Panning / Gold-Found, Signal diff vs digest)
- **[skills/work-operating-model](../work-operating-model/CONTEXT.md)** — Shares Operating Model and Decision Framework domain (Five fixed interview layers, Structural questions framework)
- **[skills/world-model-diagnostic](../world-model-diagnostic/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (OB1-connected mode vs. direct-chat mode, Personalization vs discovery balance, World-model paradigm (vector database / structured ontology / signal-fidelity))
