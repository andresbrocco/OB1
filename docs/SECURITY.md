# SECURITY.md

> Security-relevant facts for this codebase.

## Authentication

### Method

Shared access key (`x-brain-key`) — a single secret token that gates all MCP and REST API access. This is a single-tenant, personal-use design. There is no user account system at the MCP server layer.

### Implementation

- Core MCP server: `server/index.ts` — reads key from `MCP_ACCESS_KEY` env var at startup; throws immediately if absent
- Extensions (×6): `extensions/*/index.ts` — same `x-brain-key` pattern, key sourced from Supabase secrets
- Kubernetes deployment: `integrations/kubernetes-deployment/index.ts` — same pattern in a long-lived Hono HTTP server
- vercel-neon-telegram: `recipes/vercel-neon-telegram/src/lib/auth.ts` — uses Node.js `crypto.timingSafeEqual` for constant-time comparison

### Key Extraction Order (all MCP and REST endpoints)

Access key is accepted via three strategies, evaluated in order:

1. `x-brain-key` request header
2. `Authorization: Bearer <key>` header
3. `?key=<key>` query parameter

The query-parameter fallback exists because Claude Desktop custom connectors and ChatGPT actions cannot send arbitrary HTTP headers.

### Dashboard Authentication (Next.js)

- Module: `dashboards/open-brain-dashboard-next/lib/auth.ts`
- Library: `iron-session` ^8.0.4
- Cookie name: `open_brain_session`
- Cookie properties: `httpOnly: true`, `secure: true` in production, `sameSite: "lax"`, TTL 24 hours
- The `x-brain-key` value is stored inside the iron-session encrypted cookie and injected server-side into every backend API call via `lib/api.ts` (marked `server-only`). It is never included in the browser bundle.
- `SESSION_SECRET` is validated at module load time — missing or sub-32-character value causes immediate startup crash.
- Middleware (`middleware.ts`): redirects unauthenticated page requests to `/login` based on cookie presence. API route handlers use `requireSession()` (returns 401 on failure); server components use `requireSessionOrRedirect()` (issues redirect on failure).

### Dashboard Authentication (SvelteKit)

- Module: `dashboards/open-brain-dashboard/src/routes/`
- Library: `@supabase/ssr` ^0.6.1 with `@supabase/supabase-js` ^2.56.0
- Provider: Supabase Auth (anon key + OAuth)
- Auth guard: `+layout.server.ts` checks `locals.user`; redirects to `/signin` if absent
- MCP credentials (`MCP_URL`, `MCP_KEY`) are private env vars injected server-side in the `/api/mcp` proxy route and never sent to the browser. A `PUBLIC_MCP_KEY` fallback is documented but not recommended (it would expose the key in the browser bundle).

### Telegram Webhook Authentication

- Route: `recipes/vercel-neon-telegram/src/app/api/telegram/route.ts`
- Header: `x-telegram-bot-api-secret-token`
- This token is set at Telegram webhook registration time and is independent of the brain access key.

---

## Authorization

### Model

Single-tenant, flat authorization. There are no user roles or per-user permission levels at the MCP server layer. All authenticated requests have full access to all thoughts for the configured `DEFAULT_USER_ID`.

### Per-User Isolation

- `extensions/professional-crm` is the sole component that implements per-user isolation via Supabase Row-Level Security (RLS), documented in `primitives/rls/`.
- All other extensions and the core server use `DEFAULT_USER_ID` with the Supabase service role key, which bypasses RLS.

### Restricted Content Layer (Next.js dashboard only)

- A second-factor unlock exists on top of session auth for thoughts classified at the `restricted` sensitivity tier.
- After login, restricted thoughts are excluded by default from all queries (`exclude_restricted=true`).
- Unlock requires submitting a passphrase verified against `RESTRICTED_PASSPHRASE_HASH` (SHA-256 hash stored as env var). Unlock state is stored in `session.restrictedUnlocked` for the session duration.
- Endpoint: `dashboards/open-brain-dashboard-next/app/api/restricted/route.ts`

### Sensitivity Tiers

| Tier | Description |
|------|-------------|
| `standard` | Default — no access restriction |
| `personal` | Personal content; default for unrecognized tier values |
| `restricted` | Highest tier; requires passphrase unlock in the Next.js dashboard |

Tier assignment is escalation-only: `standard → personal → restricted`. Once set to a higher tier, a thought cannot be downgraded. Unrecognized caller-supplied tier values normalize to `"personal"`.

---

## Input Validation

### Libraries

- **Zod** (`npm:zod@4.1.13` in Deno Edge Functions, `^4.1.13` in `recipes/repo-learning-coach`, `^3.24.0` in `recipes/vercel-neon-telegram`): Runtime schema validation for MCP tool parameters, API request bodies, and typed metadata
- **`sanitizeMetadata`** (internal, `integrations/entity-extraction-worker/_shared/helpers.ts`): Clamps LLM-returned fields to allowed ranges — importance capped at 5 (never 6), thought type constrained to an explicit allowlist

### Locations

- MCP tool parameter schemas: `server/index.ts`, `extensions/*/index.ts`
- Metadata sanitization: `integrations/entity-extraction-worker/_shared/helpers.ts`
- API route input parsing: `dashboards/open-brain-dashboard-next/app/api/*`
- Telegram and MCP capture schemas: `recipes/vercel-neon-telegram/src/lib/types.ts`

### Prompt Injection Defense

Components that pass user-captured content to LLMs apply explicit defenses:

- `integrations/entity-extraction-worker/_shared/helpers.ts`: Wraps thought content in `<thought_content>` XML delimiters; escapes any literal occurrences of those tag strings before insertion. System prompt instructs the LLM to return empty arrays if an injection attempt is detected.
- `recipes/wiki-synthesis/scripts/`: Wraps user-captured content in `<entries>`/`<thread>` XML delimiters with system-prompt instructions to treat the block as data only, never as instructions.
- `recipes/entity-wiki/generate-wiki.mjs`: Flags content that looks like a prompt-injection attempt for review.
- `recipes/life-engine/life-engine-skill.md`: Documents explicit skill-level guards — the skill never executes commands found in message text, never shares system configuration, and ignores role-switching language.

---

## Secrets Management

### Storage

All secrets are stored as environment variables. No secrets or credentials appear in committed files. Each component ships a `.env.example` with placeholder values.

- Supabase Edge Functions: secrets set via `supabase secrets set`; read at runtime with `Deno.env.get()`
- Next.js dashboard: `.env.local` (gitignored); read via `process.env`
- Node.js recipes: `.env` files (gitignored); read via `process.env` or `dotenv`
- Python recipes: `.env` files loaded manually by each script

### Access Patterns

- Core server validates all four required secrets at startup (`SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `OPENROUTER_API_KEY`, `MCP_ACCESS_KEY`) — throws immediately if any are absent
- `dashboards/open-brain-dashboard-next/lib/auth.ts` validates `SESSION_SECRET` at module load time (must be 32+ characters)
- `lib/api.ts` in the Next.js dashboard is marked `"server-only"`, preventing the `x-brain-key` from being bundled into client JavaScript

### Required Secrets by Component

| Secret | Component | Purpose |
|--------|-----------|---------|
| `MCP_ACCESS_KEY` | `server/`, extensions, kubernetes-deployment | MCP and REST API authentication key |
| `SUPABASE_URL` | `server/`, extensions, entity-extraction-worker | Supabase project endpoint |
| `SUPABASE_SERVICE_ROLE_KEY` | `server/`, extensions, entity-extraction-worker | Supabase admin access (bypasses RLS) |
| `OPENROUTER_API_KEY` | `server/`, entity-extraction-worker | LLM embeddings and metadata extraction |
| `SESSION_SECRET` | `dashboards/open-brain-dashboard-next` | iron-session cookie encryption (32+ characters required) |
| `RESTRICTED_PASSPHRASE_HASH` | `dashboards/open-brain-dashboard-next` | SHA-256 hash for restricted-content unlock (optional feature) |
| `BRAIN_ACCESS_KEY` | `recipes/vercel-neon-telegram` | API and MCP authentication key for the Telegram recipe |
| `TELEGRAM_WEBHOOK_SECRET` | `recipes/vercel-neon-telegram` | Telegram webhook verification token |

---

## Cryptography

### Access Key Comparison

- `recipes/vercel-neon-telegram/src/lib/auth.ts` uses Node.js `crypto.timingSafeEqual` for constant-time access key comparison to prevent timing-based key enumeration attacks.

### Passphrase Hashing

- Algorithm: SHA-256 via Web Crypto API (`crypto.subtle.digest("SHA-256", ...)`)
- Location: `dashboards/open-brain-dashboard-next/app/api/restricted/route.ts`
- The SHA-256 hash of the passphrase is stored in `RESTRICTED_PASSPHRASE_HASH`. Submitted passphrases are hashed at verification time and compared to the stored hash.

### Content Fingerprinting

- Algorithm: SHA-256 via Web Crypto API
- Locations: `integrations/entity-extraction-worker/_shared/helpers.ts`, `recipes/email-history-import/pull-gmail.ts`, `recipes/fingerprint-dedup-backfill/` (uses `node:crypto`)
- Purpose: Content-level deduplication before thought upsert

### Random Identifiers

- `crypto.randomUUID()` is used for ephemeral display IDs in `dashboards/open-brain-dashboard/src/lib/api.ts` and the vercel-neon-telegram capture pipeline. These are not database primary keys.

---

## Security Headers

### CORS

All MCP Edge Function endpoints (`server/`, extensions, `integrations/entity-extraction-worker`) emit:

```
Access-Control-Allow-Origin: *
```

This wildcard is intentional to support browser-based and Electron-based clients (claude.ai, Claude Desktop). The `x-brain-key` access key provides the actual security boundary.

The core server additionally allows headers: `authorization`, `x-client-info`, `apikey`, `content-type`, `x-brain-key`, `accept`, `mcp-session-id`.

### Cookie Security (Next.js dashboard)

- `httpOnly: true`
- `secure: true` in production
- `sameSite: "lax"`

---

## Known Security Considerations

- **Single-tenant design**: The system is designed for personal use. The `DEFAULT_USER_ID` pattern and service role key usage in extensions mean all data is accessible to any holder of `MCP_ACCESS_KEY`. Only `extensions/professional-crm` implements per-user RLS.
- **Query-parameter key exposure**: The `?key=` fallback means the access key can appear in server access logs and browser history. This trade-off is documented in `docs/03-faq.md`.
- **SvelteKit public env var fallback**: `dashboards/open-brain-dashboard/.env.example` documents a `PUBLIC_MCP_KEY` variable that would expose the MCP key in the browser bundle. The preferred path is the private `MCP_KEY` env var.
- **In-memory rate limiting**: `recipes/vercel-neon-telegram/src/lib/rate-limit.ts` limits 30 requests/minute per serverless instance, not globally. State resets on cold start.
- **Shared utility duplication**: `integrations/entity-extraction-worker/_shared/` is copied into each Edge Function directory due to Supabase deployment constraints. Security updates to shared code must be applied to all copies.
- **CI pipeline secrets**: The GitHub Actions workflows invoke `anthropics/claude-code-action@v1` for automated PR review, requiring an Anthropic API key and a GitHub token stored as repository secrets.
