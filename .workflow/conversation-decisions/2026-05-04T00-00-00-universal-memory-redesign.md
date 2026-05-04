# Conversation Decisions: Universal Memory System Redesign

> Captured: 2026-05-04T00:00:00Z
> Source: conversation context (inline capture)

## Summary

Evaluated the existing OB1 (Open Brain) data model philosophically, identified that its fixed taxonomies and classification columns encode one person's worldview, and iteratively redesigned it into a universal memory substrate. The new design handles any modality (text, audio, image, documents), uses freeform labels instead of enums, and makes every freeform field semantically searchable via embeddings.

## Decisions

### Use a topological approach over a taxonomic one

- **Domain**: Design Philosophy
- **Status**: Decided
- **Value**: Stop classifying, start connecting. Capture meaning and connections, let structure emerge from the network. No pre-defined bins or fixed type hierarchies.
- **Rationale**: Taxonomies are brittle and encode one person's values. A network approach adapts to whatever content is thrown at it regardless of domain.
- **Alternatives considered**: User-configurable schemas (rejected as unnecessary complexity if naming is generic enough); hardcoded enums (rejected as too narrow)

### Build the most generalist system possible

- **Domain**: Design Philosophy
- **Status**: Decided
- **Value**: The system must accommodate different domains, different users, different content types, and different needs without requiring reconfiguration.
- **Rationale**: The original OB1 design was shaped by one person's knowledge-worker workflow and doesn't generalize to musicians, researchers, cooks, therapists, etc.

### Final schema: 6 tables (entries, nodes, links, mentions, entry_links, vocab)

- **Domain**: Data Model
- **Status**: Decided
- **Value**: The complete schema is: `entries` (universal content store with self-referential parent-child), `nodes` (knowledge graph vertices), `links` (node-to-node edges), `mentions` (entry-to-node connections), `entry_links` (entry-to-entry connections), `vocab` (semantic index of repeated freeform values).

### Rename `thoughts` to `entries`

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: The core table is called `entries` because it stores any kind of content (text, image descriptions, audio transcriptions, document chunks, facts) — not just "thoughts."
- **Rationale**: "Thought" privileges one content type. "Entry" is neutral and accommodates any modality.

### Core `entries` table structure

- **Domain**: Data Model
- **Status**: Decided
- **Value**: Columns: `id` (UUID PK), `parent_id` (UUID FK self-ref, NULL for standalone), `content` (TEXT NOT NULL, always the text representation), `embedding` (vector(1536)), `source_url` (TEXT, URL to original file in object storage), `media_type` (TEXT, MIME type), `position` (INT, ordering within parent), `fingerprint` (TEXT UNIQUE, SHA-256 dedup), `source` (TEXT, provenance), `private` (BOOLEAN DEFAULT false), `metadata` (JSONB, freeform capture-time context), `created_at`, `updated_at`.

### Remove `type` column from core table

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: No `type` column. The content IS the type. The embedding already encodes semantic type implicitly.
- **Rationale**: Type taxonomies (observation, task, idea, reference, person_note) encode one workflow's worldview and are meaningless noise for others.

### Remove `importance` column

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: No static importance score. Relevance is contextual and dynamic — the vector similarity score at query time IS the importance, relative to what you're asking.
- **Rationale**: Important for what? A grocery list is low-importance for career decisions and high-importance for tonight's dinner.

### Remove `quality_score` column

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: No quality scoring. A raw voice transcript may be "low quality" by coherence metrics but contain the only record of a breakthrough.
- **Rationale**: No objective criteria exist for scoring thought quality. Creates false confidence.

### Replace `sensitivity_tier` with `private` boolean

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: Simple `private: boolean DEFAULT false` instead of a three-tier sensitivity system.
- **Rationale**: A boolean covers 90% of the need without pretending you can enumerate sensitivity levels in advance.

### Remove `enriched` flag

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: No `enriched` column on entries. Processing state belongs in the queue infrastructure, not the data model.

### Self-referential `parent_id` for chunking large content

- **Domain**: Data Model
- **Status**: Decided
- **Value**: Entries have a nullable `parent_id` FK pointing to the same table. Short content = one row (parent_id NULL). Long documents/audio/video = one parent row + N child rows (chunks/segments). Children have `position` for ordering.
- **Rationale**: Enables handling any content size. Search finds specific chunks; parent traversal provides full context. This is RAG built into the memory layer.

### `content` column always holds a text representation

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: Every row has a TEXT `content` field — even for non-text content. For images, it's the AI-generated description. For audio, it's the transcription segment. For PDFs, it's the extracted chunk text.
- **Rationale**: Enables uniform embedding (always text-embedding model) and uniform search across all modalities.

### `source_url` for binary/large content in object storage

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: Original binary content (images, audio files, PDFs) stored in object storage, referenced by URL. NULL for plain text entries where `content` IS the original.

### `media_type` for modality identification

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: MIME-type string (e.g., "image/jpeg", "audio/mpeg", "application/pdf"). NULL for plain text. Used by ingestion pipeline to select the appropriate modality adapter.

### Rename `entities` to `nodes` with freeform `kind`

- **Domain**: Knowledge Graph
- **Status**: Decided
- **Value**: `nodes` table with columns: `id` (BIGSERIAL PK), `name` (TEXT), `normalized_name` (TEXT, dedup key), `kind` (TEXT, freeform), `description` (TEXT, one-liner), `embedding` (vector(1536), composite of name+kind+description), `aliases` (JSONB), `first_seen_at`, `last_seen_at`.
- **Rationale**: Freeform `kind` allows "composer", "spice", "emotion", "theorem", "chord progression" — whatever fits the domain — without a closed enum.
- **Alternatives considered**: No type at all (rejected — disambiguation requires some categorization); closed enum of 6 types (rejected as too narrow)

### Rename `edges` to `links` with freeform `label`

- **Domain**: Knowledge Graph
- **Status**: Decided
- **Value**: `links` table with: `id` (BIGSERIAL PK), `from_node_id` (BIGINT FK), `to_node_id` (BIGINT FK), `label` (TEXT, freeform verb/phrase), `weight` (INT DEFAULT 1, observation count), `created_at`, `updated_at`.
- **Rationale**: Freeform labels allow "composed", "treats", "tastes like", "inspired by", "is symptom of" — any natural language relationship. No enum can anticipate all domains.
- **Alternatives considered**: Fixed enum (co_occurs_with, works_on, uses, etc.) — rejected as too narrow

### Simplify `thought_entities` to `mentions`

- **Domain**: Knowledge Graph
- **Status**: Agreed (agent-suggested)
- **Value**: `mentions` table with only: `entry_id` (UUID FK), `node_id` (BIGINT FK), `created_at`. No `mention_role`, no `confidence`, no `evidence`.
- **Rationale**: The entry's content already contains the context of how the node was mentioned. Extra columns add complexity without actionable value.

### Rename `thought_edges` to `entry_links` with freeform `label`

- **Domain**: Knowledge Graph
- **Status**: Agreed (agent-suggested)
- **Value**: `entry_links` table with: `id` (BIGSERIAL PK), `from_entry_id` (UUID FK), `to_entry_id` (UUID FK), `label` (TEXT, freeform), `weight` (INT DEFAULT 1), `created_at`.
- **Rationale**: Same principle as node links — freeform labels accommodate any semantic relationship between entries.

### No confidence scores on links

- **Domain**: Knowledge Graph
- **Status**: Agreed (agent-suggested)
- **Value**: No `confidence` column on links, mentions, or entry_links. You either extracted the connection or you didn't.
- **Rationale**: LLM confidence scores are poorly calibrated. A 0.7 doesn't mean anything actionable.

### No decay_weight — handle temporal relevance at query time

- **Domain**: Knowledge Graph
- **Status**: Agreed (agent-suggested)
- **Value**: No `decay_weight` column. Temporal relevance is handled at query time (e.g., "prioritize recent") rather than as a stored field requiring cron maintenance.

### Single text-embedding model for all modalities

- **Domain**: Embedding Strategy
- **Status**: Agreed (agent-suggested)
- **Value**: Use one text embedding model (text-embedding-3-small, 1536 dims) for everything. All content is converted to text before embedding.
- **Rationale**: Keeps schema simple and embedding model uniform. Multimodal embeddings (CLIP etc.) can be added later as a second column without redesign.

### Node embedding is composite of name + kind + description

- **Domain**: Embedding Strategy
- **Status**: Agreed (agent-suggested)
- **Value**: `nodes.embedding` encodes the concatenation of the node's name, kind, and description. Makes the entire graph searchable by meaning.

### Every freeform column must be semantically searchable

- **Domain**: Embedding Strategy
- **Status**: Decided
- **Value**: All freeform text fields across all tables are searchable by semantic similarity — not just exact match.

### Vocab table for repeated freeform values (replaces label_vocab)

- **Domain**: Embedding Strategy
- **Status**: Decided
- **Value**: A single `vocab` table with columns: `id` (BIGSERIAL PK), `field` (TEXT — which column, e.g., "node.kind", "link.label", "entry.source"), `value` (TEXT — the actual string), `embedding` (vector(1536)). UNIQUE(field, value). Populated lazily — embed once per unique value.
- **Rationale**: Efficient implementation of "embed everything." 5,000 nodes of kind "person" share one embedding in vocab rather than 5,000 identical inline vectors.

### Inline embeddings for unique-per-row content, vocab for repeated values

- **Domain**: Embedding Strategy
- **Status**: Decided
- **Value**: Two embedding strategies depending on column cardinality. Unique-per-row fields (entries.content, nodes.description) get inline embedding on the row. Repeated-value fields (nodes.kind, links.label, entries.source) use the shared vocab table.
- **Rationale**: Balances full semantic searchability with storage efficiency.

### Semantic filtering via vocab table lookups

- **Domain**: Embedding Strategy
- **Status**: Decided
- **Value**: To filter by a freeform field (e.g., "find nodes of kind 'scientist'"), embed the query term, search the vocab table for semantically similar values in that field, then filter rows by those values. This catches "physicist", "biologist", "chemist" when searching for "scientist."

### Use node embedding for soft normalization/dedup at write time

- **Domain**: Knowledge Graph
- **Status**: Agreed (agent-suggested)
- **Value**: Before creating a new node, check for existing nodes with similar embeddings. If a match exists above a threshold, merge instead of duplicating. Handles "J.S. Bach" vs "Johann Sebastian Bach" and "composer" vs "musician" for the same entity.

### Generic extraction prompt (concepts + connections, no fixed schema)

- **Domain**: Ingestion Pipeline
- **Status**: Agreed (agent-suggested)
- **Value**: The extraction prompt asks for: (1) key concepts/entities mentioned (any kind, with a name and one-word kind), (2) relationships between them (natural verb phrases), (3) optionally a one-sentence summary. No fixed fields like people/action_items/dates/topics/type.
- **Rationale**: Adapts naturally to any domain. A cooking thought yields spice nodes; a music thought yields composer nodes; a therapy thought yields pattern nodes. The LLM's general knowledge handles categorization without guardrails.

### Modality adapter pipeline

- **Domain**: Ingestion Pipeline
- **Status**: Agreed (agent-suggested)
- **Value**: Ingestion follows: original content -> modality adapter (varies: passthrough for short text, chunking for long text, vision model for images, Whisper for audio, text extraction for PDFs, LLM for structured data) -> text representation -> embed -> store -> extract graph. Only the adapter step varies by content type; everything downstream is uniform.

### Facts don't need special treatment

- **Domain**: Data Model
- **Status**: Agreed (agent-suggested)
- **Value**: A fact is just an entry whose content is an assertive statement. No separate table, no special flag. The system doesn't need to distinguish facts from opinions at the storage level.
- **Rationale**: Whether something is a "fact" vs. "opinion" vs. "plan" is a semantic property captured by the embedding, not a structural one.

### Keep content fingerprint for deduplication

- **Domain**: Data Model
- **Status**: Decided
- **Value**: SHA-256 hash of normalized content for dedup. Prevents duplicate entries across import sources and repeated captures.
