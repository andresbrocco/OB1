# CONTEXT.md — Docs

## Purpose

Provides all end-user-facing documentation for the Open Brain system: initial setup guides, executable companion prompts, troubleshooting references, AI-assisted setup instructions, and operational optimization guides. This is the canonical onboarding and support surface for people building on Open Brain.

## Responsibility Boundaries

- **Owns**: User-facing setup instructions, companion prompts, FAQ content, MCP optimization guidance, and supplementary spreadsheet resources (credential tracker, platform guides)
- **Delegates to**: The `extensions/`, `primitives/`, and `recipes/` directories for implementation-level content and reusable components
- **Does not handle**: Automated workflow documentation, code-level API references, or contributor/maintainer guides (those live in `CONTRIBUTING.md` and `.github/`)

## Key Concepts

**Progressive onboarding sequence**: The numbered files (`01` through `05`) form a deliberate linear path — core setup → prompt activation → FAQ → AI-assisted setup → optimization. The numbering signals intended read order, not just alphabetical organization.

**Executable companion prompts**: `02-companion-prompts.md` is not explanatory documentation — it contains verbatim prompt templates intended to be copy-pasted into AI clients. The prompts are parameterized by the user's actual MCP connection state and vary by which AI platform the user is on.

**MCP tool context overhead**: `05-tool-audit.md` exists because a non-obvious runtime problem emerges as users accumulate extensions — each MCP tool definition consumes 150–400 tokens of context window on every message. This guide addresses routing degradation and context bloat that isn't visible in any single extension's documentation.

**Query-parameter auth pattern**: `03-faq.md` documents that Claude Desktop and ChatGPT cannot send custom headers, so the access key must be embedded as a query parameter (`?key=...`) rather than as an `Authorization` header. This distinction is easy to miss and is the most common setup failure.

## Non-Obvious Details

- The `.xlsx` files are downloadable resources (credential tracker, platform-specific setup guides for Mac and Windows) linked from the Markdown guides. They are not generated — they are static companion files maintained manually.
- `video-walkthrough-script.md` is a production script for the official setup video, not user documentation. It includes stage directions and a note about deleting demo credentials before publishing.
- `workflow-pipeline.html` is a standalone visual reference; its purpose is not described in the surrounding Markdown files.
- The AI-assisted setup guide (`04-ai-assisted-setup.md`) explains which steps in `01-getting-started.md` can be delegated to an AI coding tool and which require manual browser interaction — a useful split that isn't obvious from the main guide alone.

## Related Modules

- **[dashboards](../dashboards/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, iron-session cookie auth, sensitivity_tier restricted content gating)
- **[dashboards/open-brain-dashboard](../dashboards/open-brain-dashboard/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, SSR auth)
- **[dashboards/open-brain-dashboard-next](../dashboards/open-brain-dashboard-next/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, iron-session cookie auth, restricted content gating, server-only API proxy, two-layer auth guard)
- **[dashboards/open-brain-dashboard-next/app/api](../dashboards/open-brain-dashboard-next/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, Restricted content unlock, Session-scoped API key forwarding)
- **[dashboards/open-brain-dashboard-next/components](../dashboards/open-brain-dashboard-next/components/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, Restricted content passphrase gating)
- **[dashboards/open-brain-dashboard-next/lib](../dashboards/open-brain-dashboard-next/lib/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, restrictedUnlocked, sensitivity_tier, server-only boundary, x-brain-key)
- **[dashboards/open-brain-dashboard/src](../dashboards/open-brain-dashboard/src/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP credential server-vs-public pattern, MCP tool context overhead)
- **[dashboards/open-brain-dashboard/src/lib](../dashboards/open-brain-dashboard/src/lib/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP text-response parsing, MCP tool context overhead)
- **[dashboards/open-brain-dashboard/src/routes](../dashboards/open-brain-dashboard/src/routes/CONTEXT.md)** — Shares Authentication and Access Control domain (Auth guard via layout.server.ts, Query-parameter auth pattern)
- **[extensions](../extensions/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP tool context overhead, Per-request MCP server instantiation)
- **[extensions/family-calendar](../extensions/family-calendar/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Claude Desktop Accept-header patch, MCP tool context overhead)
- **[extensions/household-knowledge](../extensions/household-knowledge/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY pre-shared key authentication, Query-parameter auth pattern)
- **[extensions/meal-planning](../extensions/meal-planning/CONTEXT.md)** — Shares Authentication and Access Control domain (Household member RLS via JWT role claim, Query-parameter auth pattern)
- **[extensions/professional-crm](../extensions/professional-crm/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP tool context overhead, Stateless MCP transport)
- **[integrations](../integrations/CONTEXT.md)** — Shares MCP Protocol and Transport domain (Kubernetes self-hosted MCP server, MCP tool context overhead)
- **[integrations/kubernetes-deployment](../integrations/kubernetes-deployment/CONTEXT.md)** — Shares Authentication and Access Control domain (MCP_ACCESS_KEY authentication, Query-parameter auth pattern)
- **[recipes/claudeception](../recipes/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Progressive onboarding sequence, Retrospective Mode)
- **[recipes/repo-learning-coach](../recipes/repo-learning-coach/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Learning Artifacts, Progressive onboarding sequence, RepoLearningConfig)
- **[recipes/repo-learning-coach/server](../recipes/repo-learning-coach/server/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Artifact kinds (takeaway, confusion, summary), LessonStatus, Progressive onboarding sequence)
- **[recipes/repo-learning-coach/src](../recipes/repo-learning-coach/src/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, LessonStatus, Progressive onboarding sequence)
- **[recipes/repo-learning-coach/src/lib](../recipes/repo-learning-coach/src/lib/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (LearningArtifactKind, Progressive onboarding sequence)
- **[recipes/thought-enrichment](../recipes/thought-enrichment/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, Sensitivity tiers (standard/personal/restricted))
- **[recipes/vercel-neon-telegram](../recipes/vercel-neon-telegram/CONTEXT.md)** — Shares MCP Protocol and Transport domain (MCP tool context overhead, Stateless MCP transport)
- **[recipes/vercel-neon-telegram/src](../recipes/vercel-neon-telegram/src/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, timingSafeEqual auth)
- **[recipes/vercel-neon-telegram/src/app/api](../recipes/vercel-neon-telegram/src/app/api/CONTEXT.md)** — Shares Authentication and Access Control domain (Bearer token authentication, Query-parameter auth pattern, Telegram webhook secret authentication)
- **[schemas](../schemas/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, sensitivity_tier access filtering)
- **[schemas/enhanced-thoughts](../schemas/enhanced-thoughts/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, sensitivity_tier)
- **[server](../server/CONTEXT.md)** — Shares Authentication and Access Control domain (Query-parameter auth pattern, x-brain-key access key auth)
- **[skills](../skills/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Lessons Log, Progressive onboarding sequence)
- **[skills/claudeception](../skills/claudeception/CONTEXT.md)** — Shares Learning and Lesson Artifacts domain (Progressive onboarding sequence, Retrospective mode)
