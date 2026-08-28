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

## 5b. Applications table (tailored CV + ATS score + cover letter)

SQL Editor → run:

```sql
create table if not exists applications (
  id bigint generated always as identity primary key,
  job_id bigint references jobs(id) on delete cascade,
  track text not null check (track in ('industry','europe','phd')),
  cv_content text not null,
  cover_letter_content text not null,
  ats_score int,
  ats_notes text,
  created_at timestamptz default now()
);

alter table applications enable row level security;

create policy "Authenticated read" on applications
  for select using (auth.role() = 'authenticated');
create policy "Authenticated insert" on applications
  for insert with check (auth.role() = 'authenticated');
create policy "Authenticated update" on applications
  for update using (auth.role() = 'authenticated') with check (auth.role() = 'authenticated');
create policy "Authenticated delete" on applications
  for delete using (auth.role() = 'authenticated');
```

Unlike `jobs`, `id` here is a real auto-incrementing identity column — no need to compute the next id yourself on insert.

**Also run this** — new tables created via the SQL editor don't automatically get Supabase's baseline role grants the way `jobs` did (it predates this table and already had them). Without this, every request gets a `permission denied for table applications` error before RLS even applies:

```sql
grant select, insert, update, delete on public.applications to anon, authenticated, service_role;
grant usage, select on all sequences in schema public to anon, authenticated, service_role;
```

## 4b. Multiple profiles (separate job trackers per person)

Still in SQL Editor — adds a `profile` column so more than one person's tracked jobs/applications can share this one Supabase project without mixing lists. `not null default 'xavier'` backfilled every existing row to `xavier` automatically at the time; a new profile's list starts genuinely empty until rows get inserted tagged with its name. **2026-08-28: the `xavier` profile was renamed to `uk`** (all existing `jobs`/`applications` rows updated, and the column default changed to `'uk'` via `alter table jobs alter column profile set default 'uk'` / same for `applications`) — `europe` was also split out as its own profile from what used to be `xavier`'s `europe` track, and `graduate-roles` added as a new profile. See `profile/uk/background.md`, `profile/europe/background.md`, `profile/graduate-roles/background.md`.

```sql
alter table jobs
  add column if not exists profile text not null default 'uk';

alter table applications
  add column if not exists profile text not null default 'uk';
```

`/find-jobs` and `/tailor-application` read the active profile from `.active-profile` (see `/switch-profile`) and tag every row they insert with it. `index.html` has a "Who's using this?" profile picker driven by a single `PROFILES` array (id/avatar/label/title/tab/tracks) that renders the picker buttons and branding — add a new entry there for any profile beyond `uk`/`shamna`/`phd`/`europe`/`graduate-roles`/`daniel-may`.

## 4c. Deadline sorting

Still in SQL Editor — adds a `deadline` column so roles with a real application deadline can be sorted soonest-first in the tracker ("Sort: Deadline (soonest)" in `index.html`), with red/amber highlighting inside 7/21 days. Nullable — most listings don't state a hard deadline, and rows without one just sort to the bottom.

```sql
alter table jobs
  add column if not exists deadline date;
```

No RLS/grant changes needed — it's covered by the existing `jobs` policies and grants.

## 5. Service role key for local agent commands

Settings → API → copy the **service_role** key (not anon). Paste it into `.env.local` in this repo (already gitignored — never commit it):

```
SUPABASE_SERVICE_ROLE_KEY=<paste here>
```

This key is only used by `/find-jobs` and `/tailor-application` running locally in Claude Code — it never goes into `index.html` or the public site.
