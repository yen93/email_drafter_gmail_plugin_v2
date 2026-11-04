# CLAUDE.md — email_drafter_gmail_plugin_v2

Working notes for a future Claude session. For a current-state inventory see
`as_built.txt`; for session history see `chat_history/`.

## What this project is
Two halves of an email-drafter that mimics James Castrission:
1. **Ingest** — an n8n workflow caches James's recent Gmail (SENT/DRAFT) into a
   Supabase table.
2. **Draft** — a Gmail add-on (Google Apps Script) reads those cached samples live
   and uses OpenAI (gpt-4o) to draft replies in his voice from the compose box.

The repo holds COPIES to paste into their real homes, plus the table design and
reference/diagnostic files. The LIVE code does not run from this repo:
- n8n workflow export(s) → imported into n8n.
- "MAG Gmail Assistant - code gs file.txt" → the add-on's `code.gs` (Apps Script).
- "MAG Gmail Assistant - appscript json file.txt" → its `appsscript.json` manifest.

## The Gmail add-on (Apps Script)
- `handleAiDraftGeneration` → `getStyleSamplesFromDB()` (Supabase PostgREST GET,
  publishable key, recent SENT/DRAFT snippets) → `callOpenAI()` (gpt-4o).
- **Secrets:** `OPENAI_API_KEY` lives in Apps Script **Script Properties**, never
  hardcoded. `SUPABASE_URL` + the publishable (anon) key are in-code constants —
  safe, because the publishable key only reads via the anon SELECT policy.
- **Manifest gotcha:** `appsscript.json` has a `urlFetchWhitelist`. When present it
  blocks every outbound URL not listed, with "URL has not been whitelisted in the
  script manifest." It MUST list both `https://aivitcomiywiysrfwqxt.supabase.co/`
  and `https://api.openai.com/` (https, trailing slash). Needs scopes
  `script.external_request` + `gmail.modify`.
- **Deploy = paste + save in the Apps Script editor**, then reload Gmail. No
  add-on reinstall for code edits. Re-auth only prompts on NEW scopes (a whitelist
  change is not a scope). If a change won't take, the install may be pinned to a
  version deployment — bump it, or Uninstall/Install the test deployment.
- **Data limit:** the DB stores no email bodies (`payload` is just
  `{"mimeType":...}`), so `snippet` (~200 chars) is the only style signal. For
  fuller mimicry the n8n side would need to also cache decoded body text.

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
- RLS is ON. n8n writes with the **service_role** key (bypasses RLS).
  **MAGTestProject** also has policy `anon_read_james_email_samples` (SELECT only,
  `TO anon`) so the add-on reads with the publishable key. **n8n-production has NO
  such policy** (left closed deliberately) — if you ever point a reader there, add
  one. Keep this in mind if syncing the two projects: this read policy is NOT in
  n8n-production.

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
