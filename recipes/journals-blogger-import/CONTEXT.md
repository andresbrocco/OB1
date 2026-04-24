# CONTEXT.md — Journals/Blogger Import

## Purpose

Imports blog posts and comments from Google Blogger Atom XML export files into Open Brain as searchable thoughts with vector embeddings. Handles bulk historical imports from one or more `.atom` files in a directory tree.

## Responsibility Boundaries

- **Owns**: Parsing Blogger's Atom XML format, stripping HTML to plain text, generating content fingerprints for deduplication, and upserting records via the `upsert_thought` RPC.
- **Delegates to**: OpenRouter API for embedding generation, Supabase for persistence and deduplication logic.
- **Does not handle**: Incremental sync, OAuth/live API access to Blogger, or any post-import enrichment.

## Key Concepts

- **Atom XML export**: Blogger's native export format. This recipe expects a directory of `.atom` files (not direct Blogger API access). Users must manually export from Blogger settings first.
- **Entry kinds**: Blogger Atom files contain mixed entry types — `post`, `comment`, and internal kinds like `settings` and `template`. The parser filters to only `post` and `comment` entries by checking the `term` attribute of the `<category>` element against the `http://schemas.google.com/blogger/2008/kind#` namespace.
- **Content fingerprint**: A SHA-256 hash of normalized (lowercased, whitespace-collapsed) content, passed as metadata to support deduplication at the RPC level.

## Non-Obvious Details

- The Atom parser is regex-based (not a proper XML/DOM parser). It splits on `<entry>` tags and extracts fields with individual regexes. This works for well-formed Blogger exports but may silently drop entries if the XML is malformed or contains nested `<entry>` elements.
- HTML stripping is also regex-based. Embedded `<script>` or `<style>` tags in post content will have their tags removed but their text content retained.
- Content is truncated to 30,000 characters before storage and 8,000 characters before embedding. Posts exceeding these limits are silently truncated; the truncation is noted with `[... truncated]` in stored content but not in the embedding input.
- The script walks directories up to depth 3 to find `.atom` files, which accommodates multi-blog exports organized in subdirectories.
- Imported entries are stored with `type: "journal"` and `source_type: "blogger_import"`, with a fixed `importance: 3` and `quality_score: 60`.

## Related Modules

- **[.github](../../.github/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Idempotent PR comment via ob1-automated-review marker)
- **[.github/workflows](../../.github/workflows/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Idempotent PR comments)
- **[integrations/entity-extraction-worker](../../integrations/entity-extraction-worker/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, re-extraction idempotency)
- **[recipes/bring-your-own-context](../bring-your-own-context/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Portable Bundle)
- **[recipes/chatgpt-conversation-import](../chatgpt-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Sync log)
- **[recipes/email-history-import](../email-history-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Sync log, Two-layer dedup)
- **[recipes/fingerprint-dedup-backfill](../fingerprint-dedup-backfill/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint, Content fingerprint for deduplication, Duplicate row)
- **[recipes/google-activity-import](../google-activity-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Day-hash dedup via sync log)
- **[recipes/grok-export-import](../grok-export-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint deduplication, Content fingerprint for deduplication)
- **[recipes/instagram-import](../instagram-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint (SHA-256 deduplication), Content fingerprint for deduplication)
- **[recipes/life-engine](../life-engine/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Briefing deduplication, Content fingerprint for deduplication)
- **[recipes/live-retrieval](../live-retrieval/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Session-scoped deduplication)
- **[recipes/local-ollama-embeddings](../local-ollama-embeddings/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Multi-format input (stdin, positional args, .txt, .jsonl))
- **[recipes/obsidian-vault-import](../obsidian-vault-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Dual deduplication (sync log + content fingerprint))
- **[recipes/panning-for-gold](../panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Gold-Found, Panning)
- **[recipes/perplexity-conversation-import](../perplexity-conversation-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Local sync log deduplication)
- **[recipes/research-to-decision-workflow](../research-to-decision-workflow/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Skip rules)
- **[recipes/schema-aware-routing](../schema-aware-routing/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), First-person intent gate)
- **[recipes/typed-edge-classifier](../typed-edge-classifier/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Idempotent upsert via thought_edges_upsert RPC)
- **[recipes/wiki-synthesis](../wiki-synthesis/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Thread eligibility (content-weight gating))
- **[recipes/wiki-synthesis/scripts](../wiki-synthesis/scripts/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Thread eligibility gate)
- **[recipes/work-operating-model-activation](../work-operating-model-activation/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Export Artifacts (USER.md, SOUL.md, HEARTBEAT.md))
- **[recipes/x-twitter-import](../x-twitter-import/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Content fingerprinting)
- **[schemas/enhanced-thoughts](../../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, idempotent schema migration)
- **[skills/claudeception](../../skills/claudeception/CONTEXT.md)** — Shares Deduplication and Fingerprinting domain (Content fingerprint for deduplication, Open Brain deduplication workflow)
- **[skills/heavy-file-ingestion](../../skills/heavy-file-ingestion/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Converter preference (auto/native/markitdown))
- **[skills/heavy-file-ingestion/scripts](../../skills/heavy-file-ingestion/scripts/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, ConversionResult, export bundles)
- **[skills/panning-for-gold](../../skills/panning-for-gold/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Panning / Gold-Found)
- **[skills/weekly-signal-diff](../../skills/weekly-signal-diff/CONTEXT.md)** — Shares Noise Filtering and Signal Quality domain (Entry kind filtering (post/comment vs settings/template), Signal diff vs digest)
- **[skills/work-operating-model](../../skills/work-operating-model/CONTEXT.md)** — Shares Import and Export Pipelines domain (Atom XML export, Export artifacts (USER.md, SOUL.md, HEARTBEAT.md))
