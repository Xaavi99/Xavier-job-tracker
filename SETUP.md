# One-time Supabase setup

Do these once in the Supabase dashboard for project `nthonxtycsyplkmpwjrh`. Nothing here is secret except the service role key at the end — keep that out of git.

## 1. Create your login

Authentication → Users → **Add user** → enter your email + a password → toggle **Auto Confirm User** on (so you don't need to click an email link) → Create user.

This becomes the only account allowed to read/write the tracker. Use these credentials to sign in on the live site.

## 2. Lock down the `jobs` table

SQL Editor → New query → run:

The table currently has permissive policies ("Allow select/insert/update/delete", each `using (true)` — always allow, no auth check) from initial setup. These must be dropped, not just supplemented — Postgres OR's all permissive policies together, so leaving them in place means the wide-open ones keep winning even after adding authenticated-only ones.

```sql
alter table jobs enable row level security;

drop policy if exists "Allow select" on jobs;
drop policy if exists "Allow insert" on jobs;
drop policy if exists "Allow update" on jobs;
drop policy if exists "Allow delete" on jobs;
drop policy if exists "Allow all" on jobs;

create policy "Authenticated read" on jobs
  for select
  using (auth.role() = 'authenticated');

create policy "Authenticated insert" on jobs
  for insert
  with check (auth.role() = 'authenticated');

create policy "Authenticated update" on jobs
  for update
  using (auth.role() = 'authenticated')
  with check (auth.role() = 'authenticated');

create policy "Authenticated delete" on jobs
  for delete
  using (auth.role() = 'authenticated');
```

After this, the anon key alone (no login) can no longer read or write any row — confirmed by the check in step 4.

## 3. Add columns for the new workflow

Still in SQL Editor:

```sql
alter table jobs
  add column if not exists source_url text,
  add column if not exists jd_text text,
  add column if not exists cv_path text,
  add column if not exists cover_letter_path text,
  add column if not exists discovered_at timestamptz default now(),
  add column if not exists track text not null default 'industry';

alter table jobs
  drop constraint if exists jobs_track_check;
alter table jobs
  add constraint jobs_track_check check (track in ('industry','europe','phd'));
```

`track` splits roles into three tracks: `industry` (UK), `europe` (Norway/Denmark/Netherlands only), `phd` (funded PhD/CDT studentships). All existing rows default to `industry`.

## 4. Verify the lockdown

From a terminal (no login token, just the public anon key):

```
curl "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?select=id&limit=1" -H "apikey: <anon key>"
```

Before this fix that returns a row. After it, it should return `[]` or a 401/permission error — confirming anonymous access is closed.

## 5. Service role key for local agent commands

Settings → API → copy the **service_role** key (not anon). Paste it into `.env.local` in this repo (already gitignored — never commit it):

```
SUPABASE_SERVICE_ROLE_KEY=<paste here>
```

This key is only used by `/find-jobs` and `/tailor-application` running locally in Claude Code — it never goes into `index.html` or the public site.
