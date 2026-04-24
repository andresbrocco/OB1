# CONTEXT.md — Entity Wiki

## Purpose

Generates structured markdown wiki pages for individual entities in the Open Brain knowledge graph. For each entity (person, project, topic, organization, tool, or place), the script aggregates linked thoughts and typed relationship edges, then synthesizes a formatted wiki page via an LLM using an OpenAI-compatible Chat Completions endpoint. Output can be written to local files, stored in the entity's metadata column, or upserted as a special dossier thought.

## Responsibility Boundaries

- **Owns**: Evidence aggregation from `thought_entities` and `edges` tables, LLM prompt construction and synthesis, output routing across three distinct persistence modes, slug collision resolution for file output.
- **Delegates to**: Supabase PostgREST for all data queries, an external LLM API (OpenRouter by default) for synthesis, an optional embedding provider (OpenAI by default) for semantic expansion and dossier embedding.
- **Does not handle**: Entity extraction or creation, the `thoughts` table capture flow, embedding generation for ordinary thoughts, or MCP server concerns.

## Key Concepts

- **Emergent cached view**: Wikis are derived, regenerable artifacts. `public.thoughts` remains the sole source of truth; wiki pages should be treated as a cache and never edited in place.
- **Output modes**: Three modes with distinct trade-offs — `file` writes a frontmatter-annotated `.md` file; `entity-metadata` stores the wiki under `entities.metadata.wiki_page` (zero thought-store pollution); `thought` upserts a `dossier`-typed thought (enables MCP search but pollutes semantic search unless filtered by `metadata.type = 'dossier'`).
- **Semantic expansion**: Optional flag (`--semantic-expand`) that broadens evidence by running a pgvector similarity search (`match_thoughts` RPC) against the entity name. Requires a separate embedding provider and is independent of the chat LLM.
- **co_occurs_with exclusion**: The `edges` query hard-excludes `co_occurs_with` relation edges at the SQL layer as noise. This exclusion propagates through `buildSynthesisInput` and the system prompt, which explicitly instructs the model not to render a co-mention subsection.
- **Slug collision resolution**: `slugify()` strips non-ASCII characters, so distinct entities like `C`, `C#`, and `C++` can share a base slug. `resolveOutputPath` uses frontmatter `entity_id:` inspection to distinguish idempotent re-runs (same entity, overwrite) from true collisions (different entity, append numeric suffix with a warning).

## Non-Obvious Details

- **Prompt injection defense**: User-captured thought content is untrusted. The script applies a two-layer defense: a pre-scrub (`scrubSnippetContent`) that strips control characters, neutralizes `<thought>` tag injection, and flags known injection phrases in-place; and a structural split in the user message that separates trusted JSON structure from untrusted fenced `<thought>` blocks. The system prompt explicitly instructs the model to surface injection attempts in `## Open Questions` rather than obey them.
- **Dossier idempotency**: When `--output-mode=thought`, a re-run must refresh the existing dossier row rather than accumulate duplicates. The script first queries `thoughts` by `metadata.wiki_entity_id`; if found, it PATCHes in place. The content intentionally omits the generation timestamp (which goes only in `metadata`) to avoid defeating the `upsert_thought` content-fingerprint dedup on the fallback path.
- **Embedding preflight**: Before processing any entity in `thought` or `--semantic-expand` mode, the script embeds a probe string and verifies the returned dimension matches `vector(1536)`. It also probes the `match_thoughts` RPC signature when semantic expansion is active. Both checks fail fast with actionable error messages rather than producing N silent per-entity failures.
- **Alias resolution gap**: `resolveEntityByName` matches `canonical_name` and `normalized_name` only. Entities reachable only via an alias in the JSONB `aliases` column require `--id` (PostgREST `cs` filter inside an `or=(...)` clause is noted as brittle across versions).
- **Batch candidate heuristic**: `listBatchCandidates` cannot use `GROUP BY` directly via PostgREST without an RPC, so it over-fetches entities ordered by `last_seen_at` and performs per-entity link-count probes. For large knowledge graphs this is N+1 and the script comments that users should add a custom RPC.

## Related Modules

- **[.claude/skills](../../.claude/skills/CONTEXT.md)** — Shares Prompt Injection and Security domain (Human-judgment layer, Prompt injection defense)
- **[.github](../../.github/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, security-blocked and needs-maintainer-triage label lifecycle)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, pull_request vs pull_request_target security split)
- **[dashboards/open-brain-dashboard/src/routes](../../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Incremental result merging, Slug collision resolution)
- **[extensions](../../extensions/CONTEXT.md)** — Shares Output and File Writing Discipline domain (AGENT_SPEC.md machine-readable generation spec, Output modes (file / entity-metadata / thought))
- **[extensions/job-hunt](../../extensions/job-hunt/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension CRM link, Dossier thought, Job contact vs professional contact, Pipeline (application status lifecycle))
- **[extensions/professional-crm](../../extensions/professional-crm/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Cross-extension bridge via denormalized note append, Dossier thought, Fixed opportunity stage enum, Interaction log vs. follow-up date (two separate follow-up signals), Trigger-managed last_contacted field)
- **[integrations](../../integrations/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Symmetric relation canonical ordering, co_occurs_with exclusion)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, support_count, symmetric relation canonicalization)
- **[integrations/kubernetes-deployment](../../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Dual API configuration (embedding vs. chat), Semantic expansion, pgvector cosine distance via raw SQL)
- **[integrations/kubernetes-deployment/k8s](../../integrations/kubernetes-deployment/k8s/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Semantic expansion, match_thoughts RPC equivalent)
- **[recipes](../CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Output modes (file / entity-metadata / thought), Recipe vs. Extension distinction, Recipe vs. Skill distinction, _template skeleton, metadata.json contribution contract, requires_skills delegation pattern)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Wiki and Knowledge Compilation domain (Pyramid summaries, Slug collision resolution)
- **[recipes/claudeception](../claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Output modes (file / entity-metadata / thought), Skill Lifecycle)
- **[recipes/infographic-generator](../infographic-generator/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Manifest file, Output modes (file / entity-metadata / thought))
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Hit threshold (score > 0.6), Retrieval log, Semantic expansion, Session cap (max 3 retrievals))
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Embedding dimension mismatch, Local embedding via Ollama, Semantic expansion)
- **[recipes/ob-graph](../ob-graph/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, edge weight, find_shortest_path (bidirectional BFS recursive CTE), graph_edges, graph_nodes, relationship_type, traverse_graph (outgoing recursive CTE))
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Secret scanning)
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Emergent cached view, Flywheel closure)
- **[recipes/repo-learning-coach](../repo-learning-coach/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (Brain Bridge, Emergent cached view, Understanding State)
- **[recipes/repo-learning-coach/server](../repo-learning-coach/server/CONTEXT.md)** — Shares World Model and Knowledge Paradigm domain (BrainBridge, Emergent cached view, UnderstandingState)
- **[recipes/repo-learning-coach/src](../repo-learning-coach/src/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughts, Semantic expansion)
- **[recipes/repo-learning-coach/src/lib](../repo-learning-coach/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (RelatedThoughtSummary, Semantic expansion)
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Output modes (file / entity-metadata / thought), Prompt stubs)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Dossier thought, Pending person confirmation)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge direction (A_to_B, B_to_A, symmetric), Typed relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), co_occurs_with exclusion)
- **[recipes/vercel-neon-telegram/src/lib](../vercel-neon-telegram/src/lib/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Semantic expansion, match_thoughts)
- **[recipes/wiki-compiler](../wiki-compiler/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Compile manifest, Output modes (file / entity-metadata / thought))
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output modes (file / entity-metadata / thought), Resume-safe JSONL state)
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, derived_from edges)
- **[schemas](../../schemas/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Reasoning relation vocabulary (supports, contradicts, evolved_into, supersedes, depends_on, related_to), Two-tier edge model (entity edges vs thought edges), co_occurs_with exclusion)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (co_occurs_with exclusion, metadata-overlap thought connections)
- **[schemas/entity-extraction](../../schemas/entity-extraction/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Edge support count, co_occurs_with exclusion)
- **[schemas/typed-reasoning-edges](../../schemas/typed-reasoning-edges/CONTEXT.md)** — Shares Graph and Knowledge Edges domain (Relation vocabulary (supports/contradicts/evolved_into/supersedes/depends_on/related_to), co_occurs_with exclusion, support_count evidence accumulation)
- **[server](../../server/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Semantic expansion, match_thoughts RPC (pgvector similarity search))
- **[skills](../../skills/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output Contract, Output modes (file / entity-metadata / thought))
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Dual-scope skill storage, Output modes (file / entity-metadata / thought), Skill lifecycle (creation to archival))
- **[skills/deal-memo-drafting](../../skills/deal-memo-drafting/CONTEXT.md)** — Shares CRM and Professional Contact Tracking domain (Diligence packet, Dossier thought)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output directory convention (.ob1/), Output modes (file / entity-metadata / thought))
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Output and File Writing Discipline domain (.ob1 output directory, Output modes (file / entity-metadata / thought))
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Output and File Writing Discipline domain (Output modes (file / entity-metadata / thought), Permanent file write discipline)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Vector Search and Retrieval domain (Live search upgrade, Semantic expansion)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Skill and Recipe Contribution Framework domain (Canonical entry contract, Output modes (file / entity-metadata / thought))
- **[skills/world-model-diagnostic](../../skills/world-model-diagnostic/CONTEXT.md)** — Shares Prompt Injection and Security domain (Prompt injection defense, Simulated judgment)
