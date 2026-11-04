# CLAUDE.md — email_drafter_gmail_plugin_v2

Working notes for a future Claude session. For a current-state inventory see
`as_built.txt`; for session history see `chat_history/`.

## What this project is
An n8n-driven cache of James Castrission's recent Gmail emails (SENT/DRAFT),
stored in Supabase, used to feed an email-drafter AI that mimics his tone/style.
There is no application code here yet — the repo holds the n8n workflow export,
the target table's design, and reference/diagnostic files.

## The Supabase table (`public.james_email_samples`)
- Lives in **MAGTestProject** (`aivitcomiywiysrfwqxt`) — the real target — and
  also in **n8n-production** (`uwsncuowtzydylfabikj`). Keep both in sync if you
  change the schema.
- **Columns are intentionally quoted/mixed-case** (`Subject`, `From`, `To`,
  `Cc`, `threadId`, `sizeEstimate`, `historyId`, `internalDate`) to match the
  Gmail JSON keys exactly. This is what lets n8n's **Auto-map input data to
  columns** work. DO NOT lowercase or rename these — auto-map would silently
  stop mapping those fields.
- `payload` and `labels` are `jsonb` (nested object / array).
- `received_at` is a GENERATED column parsed from `internalDate` (epoch-ms
  string) — use it for sorting/filtering "freshest", not the raw string.
- `created_at` defaults to `now()`. Neither generated/default column is in the
  JSON, so auto-map ignores them.
- RLS is ON with no policies: n8n's **service_role** key bypasses it. If a
  reader uses an anon/publishable key it gets nothing — add a SELECT policy then.

## The n8n workflow ("Gmail Email Samples Caching")
Schedule Trigger → Gmail "Get many messages" (getAll, limit 20, sender
`james@myadventuregroup.com.au`) → **Normalize keys** (Code node) → Supabase
"Create a row" (auto-map).

### Gotcha 1 — PGRST102 "All object keys must match" (FIXED)
The v1 Supabase node with auto-map sends ALL items as one PostgREST **bulk
insert**, which requires every object to have an identical key set. Gmail items
differ (some have `Cc`, some don't), so the batch is rejected. The **Normalize
keys** Code node fixes this by forcing every item to the same 12 keys (missing →
`null`, stray keys without a column → dropped). If you ever see PGRST102 again,
it's non-uniform keys reaching the Supabase node.

### Gotcha 2 — duplicate key 23505 on re-runs (OPEN)
The schedule re-pulls the latest 20 each run, so ids overlap, but the v1
Supabase node does a plain INSERT (no upsert). The first run succeeds; later
runs will fail on the PK. To make it idempotent, replace the Supabase node with
an **HTTP Request** node (`POST …/rest/v1/james_email_samples`, header
`Prefer: resolution=merge-duplicates`) or add a dedup step. Not yet done.

## Conventions
- Credentials (Gmail OAuth2, Supabase service_role) live in n8n only — never in
  this repo. Workflow exports contain credential *reference ids*, not secrets.
- Two Supabase projects exist for this account; when running MCP Supabase tools,
  pick the ref deliberately (MAGTestProject = aivitcomiywiysrfwqxt).
