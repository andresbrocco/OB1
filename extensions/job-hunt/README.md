# Job Hunt Pipeline

> Extension 6 of the Open Brain learning path: a Supabase Edge Function MCP server for end-to-end job search tracking — companies, postings, applications, interviews, and contacts with CRM integration.

## Quick Reference

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Supabase project URL (`https://your-project.supabase.co`) | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (set automatically on deploy) | Yes |
| `MCP_ACCESS_KEY` | Bearer key required on every MCP request (`?key=` or `x-access-key` header) | Yes |
| `DEFAULT_USER_ID` | UUID of the default user; set as an Edge Function secret | Yes |

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically by Supabase at deploy time. Only `MCP_ACCESS_KEY` and `DEFAULT_USER_ID` require manual configuration.

### HTTP Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/*` | MCP JSON-RPC endpoint — all tool calls |
| `GET` | `/*` | Health check |

Authentication: every `POST` request must include the access key as a query parameter or header.

```bash
# Health check
curl https://your-project.supabase.co/functions/v1/job-hunt

# Call an MCP tool (example: get_pipeline_overview)
curl -X POST \
  "https://your-project.supabase.co/functions/v1/job-hunt?key=YOUR_MCP_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "get_pipeline_overview",
      "arguments": { "days_ahead": 7 }
    }
  }'
```

### MCP Tools

| Tool | Description |
|------|-------------|
| `add_company` | Add a company to track in your job search |
| `add_job_posting` | Add a job posting at a tracked company |
| `add_job_contact` | Add a recruiter, hiring manager, referral, or interviewer |
| `submit_application` | Record a submitted application |
| `schedule_interview` | Schedule an interview for an application |
| `log_interview_notes` | Add post-interview feedback and mark as completed |
| `get_pipeline_overview` | Dashboard summary: status counts and upcoming interviews |
| `get_upcoming_interviews` | List interviews in the next N days with full context |
| `search_job_contacts` | Search or filter job contacts by name, role, or company |
| `link_contact_to_professional_crm` | Cross-extension: promote a job contact to Extension 5's `professional_contacts` table |

### Commands

```bash
# Deploy the Edge Function via Supabase CLI
supabase functions deploy job-hunt

# Set required secrets
supabase secrets set MCP_ACCESS_KEY=your-key
supabase secrets set DEFAULT_USER_ID=your-uuid

# Run schema migrations
supabase db push
# or apply manually:
psql "$DATABASE_URL" -f schema.sql

# Serve locally for development
supabase functions serve job-hunt --env-file .env.example
```

### Configuration

| File | Purpose |
|------|---------|
| `deno.json` | Deno import map — pins all npm dependencies |
| `.env.example` | Template showing required environment variables |
| `schema.sql` | Full database schema with RLS policies and indexes |

### Database Tables

| Table | Purpose |
|-------|---------|
| `companies` | Organizations tracked in your job search (industry, size, remote policy, Glassdoor rating) |
| `job_postings` | Specific roles at companies (salary range, requirements, source, closing date) |
| `applications` | Submitted applications with status pipeline (`draft` → `accepted`/`rejected`) |
| `interviews` | Scheduled and completed interviews with prep notes and post-interview feedback |
| `job_contacts` | Recruiters, hiring managers, referrals, and interviewers; optional link to Extension 5 CRM |

All tables have Row Level Security enabled. Users can only read and write their own rows.

Application status values: `draft`, `applied`, `screening`, `interviewing`, `offer`, `accepted`, `rejected`, `withdrawn`.

Interview type values: `phone_screen`, `technical`, `behavioral`, `system_design`, `hiring_manager`, `team`, `final`.

### Prerequisites

- Open Brain core setup complete (Supabase project with `thoughts` table)
- Supabase CLI installed
- Extension 5 (Professional CRM) deployed if you intend to use `link_contact_to_professional_crm`
- Primitives read: `deploy-edge-function`, `remote-mcp`, `rls`

## Common Tasks

### Add a company and first job posting

```
# Step 1 — add the company
add_company: { "name": "Acme Corp", "industry": "SaaS", "size": "mid-market", "remote_policy": "remote" }
# → returns company.id

# Step 2 — add a posting using the returned ID
add_job_posting: { "company_id": "<id>", "title": "Senior Engineer", "url": "https://...", "salary_min": 150000, "salary_max": 190000 }
# → returns job_posting.id

# Step 3 — submit an application
submit_application: { "job_posting_id": "<id>", "status": "applied", "applied_date": "2026-04-23" }
```

### Track an interview from scheduling through debrief

```
# Schedule
schedule_interview: {
  "application_id": "<id>",
  "interview_type": "technical",
  "scheduled_at": "2026-04-30T14:00:00Z",
  "duration_minutes": 60,
  "interviewer_name": "Jane Smith"
}
# → returns interview.id

# After the interview, log notes
log_interview_notes: {
  "interview_id": "<id>",
  "feedback": "Strong on system design, review concurrency patterns",
  "rating": 4
}
```

### Check your pipeline dashboard

```
get_pipeline_overview: { "days_ahead": 7 }
# → returns total application count, status breakdown, and upcoming interviews
```

### Promote a job contact to your Professional CRM

```
# Find the contact ID
search_job_contacts: { "query": "Jane Smith", "role_in_process": "hiring_manager" }

# Link to Extension 5
link_contact_to_professional_crm: { "job_contact_id": "<id>" }
```

### Connect Claude Desktop

In Claude Desktop: Settings → Connectors → Add custom connector → paste your deployed Edge Function URL with `?key=YOUR_MCP_ACCESS_KEY`.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| `401 Unauthorized` on every request | `MCP_ACCESS_KEY` mismatch or missing | Confirm the key in your request matches `supabase secrets set MCP_ACCESS_KEY=...` |
| `DEFAULT_USER_ID not configured` (500) | Secret not set | Run `supabase secrets set DEFAULT_USER_ID=your-uuid` and redeploy |
| `Failed to add company: ...` | Schema not applied | Run `supabase db push` or apply `schema.sql` manually |
| `link_contact_to_professional_crm` fails with CRM insert error | Extension 5 `professional_contacts` table missing | Deploy Extension 5 (Professional CRM) and apply its schema first |
| Health check returns 200 but tool calls time out | Edge Function cold start | Retry; Supabase Edge Functions spin down after inactivity |
| RLS policy violation | `DEFAULT_USER_ID` does not match `auth.uid()` in policies | The service role key bypasses RLS by default; confirm `createClient` is using the service role key |

## Related

- [CONTEXT.md](CONTEXT.md) — Architecture context for this extension
- [../professional-crm/README.md](../professional-crm/README.md) — Extension 5, required for CRM cross-linking
- [../../primitives/deploy-edge-function/README.md](../../primitives/deploy-edge-function/README.md) — Edge Function deployment guide
- [../../primitives/remote-mcp/README.md](../../primitives/remote-mcp/README.md) — Remote MCP pattern
- [../../primitives/rls/README.md](../../primitives/rls/README.md) — Row Level Security primer
