# Entity Extraction Worker

> Supabase Edge Function that drains the entity extraction queue, calling an LLM to extract named entities and relationships from thoughts and writing results into the knowledge graph.

## Quick Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SUPABASE_URL` | Supabase project URL | — | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key with write access to all tables | — | Yes |
| `MCP_ACCESS_KEY` | Shared secret used to authenticate inbound requests (`x-brain-key` header or `Authorization: Bearer`) | — | Yes |
| `OPENROUTER_API_KEY` | OpenRouter API key (primary LLM provider) | — | One of three required |
| `OPENAI_API_KEY` | OpenAI API key (secondary LLM provider) | — | One of three required |
| `ANTHROPIC_API_KEY` | Anthropic API key (tertiary LLM provider) | — | One of three required |
| `OPENROUTER_CLASSIFIER_MODEL` | Override the OpenRouter extraction model | `anthropic/claude-haiku-4-5` | No |
| `OPENAI_CLASSIFIER_MODEL` | Override the OpenAI extraction model | `gpt-4o-mini` | No |
| `ANTHROPIC_CLASSIFIER_MODEL` | Override the Anthropic extraction model | `claude-haiku-4-5-20251001` | No |
| `ENTITY_EXTRACTION_MAX_CALLS` | Max LLM calls per container cold-start (0 disables cap) | `10000` | No |
| `FETCH_TIMEOUT_MS` | Hard timeout per outbound LLM fetch in milliseconds | `60000` | No |

At least one of `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, or `ANTHROPIC_API_KEY` must be set. Provider fallback order is OpenRouter → OpenAI → Anthropic.

### Endpoints

This worker is a Supabase Edge Function invoked via HTTP. After deploying, trigger it on a schedule (e.g. Supabase cron or an external scheduler).

| Method | Query Params | Description |
|--------|-------------|-------------|
| `GET` / `POST` | `limit`, `dry_run` | Process a batch of pending queue items |
| `OPTIONS` | — | CORS preflight (returns 204) |

**Query parameters:**

| Param | Type | Default | Max | Description |
|-------|------|---------|-----|-------------|
| `limit` | integer | `10` | `50` | Number of queue items to claim and process per invocation |
| `dry_run` | boolean | `false` | — | Extract entities but skip all writes; returns extracted data in `details` |

**Authentication:** Pass the `MCP_ACCESS_KEY` value in one of three ways:
- `x-brain-key: <key>` header
- `Authorization: Bearer <key>` header
- `?key=<key>` query parameter

**Example — trigger a normal batch:**

```bash
curl -X POST \
  "https://<project-ref>.supabase.co/functions/v1/entity-extraction-worker?limit=20" \
  -H "x-brain-key: $MCP_ACCESS_KEY"
```

**Example — dry run to preview extraction without writing:**

```bash
curl -X POST \
  "https://<project-ref>.supabase.co/functions/v1/entity-extraction-worker?limit=5&dry_run=true" \
  -H "x-brain-key: $MCP_ACCESS_KEY"
```

**Example response:**

```json
{
  "processed": 5,
  "succeeded": 4,
  "failed": 0,
  "entities_created": 12,
  "edges_created": 7,
  "dry_run": false,
  "truncated": false,
  "truncated_reason": null,
  "llm_calls": 4,
  "elapsed_ms": 3421
}
```

### Commands

```bash
# Deploy to Supabase
supabase functions deploy entity-extraction-worker --no-verify-jwt

# Set required secrets
supabase secrets set MCP_ACCESS_KEY=<value>
supabase secrets set SUPABASE_URL=<value>
supabase secrets set SUPABASE_SERVICE_ROLE_KEY=<value>
supabase secrets set OPENROUTER_API_KEY=<value>

# Run locally for development (Deno required)
supabase functions serve entity-extraction-worker

# Invoke locally against local Supabase
curl -X POST "http://localhost:54321/functions/v1/entity-extraction-worker?limit=5&dry_run=true" \
  -H "x-brain-key: test-key"
```

### Configuration

| File | Purpose |
|------|---------|
| `deno.json` | Deno import map — pins `@supabase/supabase-js@2.47.10` |
| `_shared/config.ts` | Shared constants: classifier model names, field length limits, sensitivity patterns |
| `_shared/helpers.ts` | Shared utility functions: `extractMetadata`, `embedText`, `detectSensitivity`, type coercion helpers |

### Database Tables

| Table | Purpose |
|-------|---------|
| `entity_extraction_queue` | Source queue. Worker claims rows (`status=pending → processing`), then marks them `complete`, `failed`, or `skipped` |
| `entities` | Canonical entity records. Upserted on `(entity_type, normalized_name)` |
| `thought_entities` | Junction: links a thought to the entities extracted from it (`source='entity_worker'`) |
| `edges` | Knowledge graph edges between entity pairs. `support_count` increments on re-extraction |
| `thoughts` | Read-only source of thought content for extraction |

These tables are defined in `schemas/entity-extraction` (knowledge-graph schema). They must exist before deploying the worker.

### Prerequisites

- Supabase project with the knowledge-graph schema applied (`schemas/entity-extraction`)
- Supabase CLI (`supabase` v1.x or later)
- Deno runtime (for local development only)
- At least one LLM API key (OpenRouter, OpenAI, or Anthropic)

## Common Tasks

### Deploy the worker

```bash
# 1. Apply the knowledge-graph schema (first time only)
supabase db push  # or run the SQL from schemas/entity-extraction manually

# 2. Set secrets
supabase secrets set MCP_ACCESS_KEY=<your-key>
supabase secrets set OPENROUTER_API_KEY=<your-key>

# 3. Deploy
supabase functions deploy entity-extraction-worker --no-verify-jwt
```

### Trigger extraction manually

```bash
curl -X POST \
  "https://<project-ref>.supabase.co/functions/v1/entity-extraction-worker?limit=50" \
  -H "x-brain-key: $MCP_ACCESS_KEY"
```

### Set up a recurring cron

In the Supabase dashboard under **Database → Cron Jobs**, add a job:

```sql
select cron.schedule(
  'entity-extraction-worker',
  '*/5 * * * *',   -- every 5 minutes
  $$
    select net.http_post(
      url := 'https://<project-ref>.supabase.co/functions/v1/entity-extraction-worker?limit=20',
      headers := '{"x-brain-key": "<MCP_ACCESS_KEY>"}'::jsonb
    );
  $$
);
```

### Preview extractions without writing (dry run)

```bash
curl -X POST \
  "https://<project-ref>.supabase.co/functions/v1/entity-extraction-worker?limit=10&dry_run=true" \
  -H "x-brain-key: $MCP_ACCESS_KEY"
```

The response `details` array will contain each thought's extracted entities and relationships. Nothing is written to the database.

### Override the extraction model

```bash
supabase secrets set OPENROUTER_CLASSIFIER_MODEL=anthropic/claude-3-5-haiku
```

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `503 Service misconfigured: auth key not set` | `MCP_ACCESS_KEY` secret is missing | Run `supabase secrets set MCP_ACCESS_KEY=<value>` and redeploy |
| `401 Unauthorized` | Request sent without or with wrong key | Pass the correct value via `x-brain-key` header or `Authorization: Bearer` |
| `503 No LLM API key configured` | None of the three LLM keys are set | Set at least one of `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, or `ANTHROPIC_API_KEY` |
| `truncated: true, truncated_reason: "wall_clock_budget"` | Batch exceeded the 140s processing budget | Reduce `limit` (try `10`–`20`); unclaimed rows are automatically returned to `pending` |
| `truncated: true, truncated_reason: "call_cap_reached"` | `ENTITY_EXTRACTION_MAX_CALLS` hit within a single container lifetime | Increase `ENTITY_EXTRACTION_MAX_CALLS` or invoke again; cap resets on cold start |
| Queue rows stuck in `status=processing` | Worker was hard-killed by Supabase before cleanup | Manually reset: `UPDATE entity_extraction_queue SET status='pending', started_at=null WHERE status='processing'` |
| All LLM calls failing with `fetch timeout` | LLM upstream slow or unreachable | Increase `FETCH_TIMEOUT_MS`; check OpenRouter/OpenAI/Anthropic status pages |
| Entities not appearing after successful run | Knowledge-graph schema not applied | Verify `entities`, `edges`, and `thought_entities` tables exist in your Supabase project |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this worker
- [_shared/CONTEXT.md](_shared/CONTEXT.md) — Architecture context for shared helpers
- [../../schemas/entity-extraction](../../schemas/entity-extraction) — Database schema this worker depends on
- [../../integrations/README.md](../README.md) — Integrations directory overview
