---
description: Search for new matching roles/studentships across all three tracks and add them to the job tracker
argument-hint: "[industry|europe|phd] (optional — omit to run all three)"
---

Run the job discovery pipeline for Xavier's job tracker across its three tracks. Argument: $ARGUMENTS — if one of `industry`/`europe`/`phd` is given, run only that track; otherwise run all three. Do this yourself step by step — do not skip steps or ask for confirmation mid-way unless something is actually ambiguous.

## 0. Load context
- Read `.active-profile` to get `$PROFILE` (if the file is missing, `$PROFILE` is `xavier`). Every `profile/...` path below means `profile/$PROFILE/...`. If `profile/$PROFILE/` doesn't exist, stop and tell Xavier to run `/switch-profile` to pick a valid one.
- Read `profile/$PROFILE/background.md` — defines all three tracks (`industry`, `europe`, `phd`), their target roles, and locations. Read `profile/$PROFILE/cv.md` (industry/europe) and `profile/$PROFILE/cv-phd.md` (phd). If a needed CV file still contains only placeholder text, warn Xavier at the end but continue using `background.md` alone.
- Read `.env.local` for `SUPABASE_SERVICE_ROLE_KEY`. If it's empty, stop and tell Xavier to complete `SETUP.md` step 5 first.

## 1. Fetch existing tracker rows (for dedupe)
```
curl "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?select=company,role,link,track&profile=eq.$PROFILE" \
  -H "apikey: <service role key>" \
  -H "Authorization: Bearer <service role key>"
```
Filtered to `$PROFILE` — each profile's tracker is independent, so the same external role can legitimately appear in more than one person's list without counting as a duplicate. Keep `(company, role, link, track)` in memory for dedupe in step 4. Service role key bypasses RLS — local use only, never expose it in `index.html` or commit it.

## 2. Discovery subagents (one per track being run)
For each track in scope, launch a `general-purpose` Agent (fresh, not a fork) with a self-contained prompt:

**`industry`**: target roles from `profile/$PROFILE/background.md`'s `industry` section (O&M/Reliability/Asset Integrity/RBI/Condition Monitoring/Structural-FEA/adjacent Mechanical grad roles), location UK only. Search LinkedIn Jobs, Indeed, Glassdoor, EnergyJobline, RenewableUK jobs board, and relevant company career pages.

**`europe`**: same target roles, location any European country except the UK — not restricted to a fixed shortlist. Norway, Denmark, and the Netherlands are the densest offshore wind O&M hubs (Equinor, Ørsted, Dutch North Sea projects) and are worth searching first/weighting higher on fit, but also search Germany, Belgium, France, Ireland, and other European offshore wind markets — don't skip a genuinely strong match just because it's outside the original three. Search the same general boards plus relevant national boards (e.g. Finn.no, Jobindex.dk, national offshore wind operator career pages) for whichever countries turn up real roles.

**`phd`**: funded PhD/CDT studentships in offshore wind O&M cost modelling, reliability engineering, predictive maintenance, or digital twins. Search FindAPhD, jobs.ac.uk, and university engineering-department funding/studentship pages — not general job boards. Location is secondary to research fit.

Each agent returns a structured list per candidate: `role, company, location, link, source_url, jd_text` (jd_text = a few sentences summarizing actual requirements, not just the search snippet). Cap each track at ~15-25 candidates. Run all requested tracks' discovery agents in parallel (single message, multiple Agent calls).

## 3. Dedupe
Drop any candidate matching an existing row from step 1 by `link`, or by `(company, role, track)` if the link differs but both match closely.

## 4. Fit-scoring
For each remaining new candidate, compare its `jd_text` against the relevant profile file(s) — `profile/$PROFILE/background.md` + `cv.md` for `industry`/`europe`, `background.md` + `cv-phd.md` for `phd`. Assign:
- `tier`: `A` (strong direct match), `B` (adjacent/plausible fit), `C` (stretch/loosely related)
- `notes`: one grounded sentence explaining *why*, referencing Xavier's actual background (RBI/Asset Integrity experience, CTMC/MPC dissertation with real numbers, digital twin/predictive maintenance angle, FEA/Abaqus) — not generic filler.

Do this yourself directly (it's reasoning over text, not search) rather than spawning another subagent, unless the combined candidate list is large enough that batching through a subagent is clearly faster.

## 5. Insert new rows
**The `jobs.id` column has no auto-increment default** — omitting it causes a `null value in column "id"` failure. First fetch the current max id, then assign each new row the next sequential integer yourself:
```
curl "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?select=id&order=id.desc&limit=1" \
  -H "apikey: <service role key>" -H "Authorization: Bearer <service role key>"
```

For each new, scored candidate, POST to Supabase with the service role key, setting `id` to the next unused integer and `track` to the track it came from:

```
curl -X POST "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs" \
  -H "apikey: <service role key>" \
  -H "Authorization: Bearer <service role key>" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=minimal" \
  -d '{"id":<next int>,"role":"...","company":"...","location":"...","tier":"A","status":"Interested","date":"","notes":"...","link":"...","referral":false,"track":"industry","source_url":"...","jd_text":"...","discovered_at":"<ISO timestamp now>","profile":"$PROFILE"}'
```

Batching multiple candidates into a single POST with a JSON array body works too — just give each element in the array its own sequential `id`.

## 6. Report
Tell Xavier: how many new roles were added per track, broken down by tier within each, and call out the top 2-3 standout matches overall by name with a one-line reason. Remind him the live tracker will show them next time he opens/refreshes the page (after signing in), under the `$PROFILE` profile pill, filterable further by the track pills.
