---
description: Search for new matching roles/studentships across all three tracks and add them to the job tracker
argument-hint: "[industry|europe|phd] (optional — omit to run all three)"
---

Run the job discovery pipeline for the active profile's (`$PROFILE`, from `.active-profile`) job tracker across its tracks. Argument: $ARGUMENTS — if one of `industry`/`europe`/`phd` is given, run only that track; otherwise run all three that have a defined scope (see step 0 — a profile may not use all three). Do this yourself step by step — do not skip steps or ask for confirmation mid-way unless something is actually ambiguous.

## 0. Load context
- Read `.active-profile` to get `$PROFILE` (if the file is missing, `$PROFILE` is `xavier`). Every `profile/...` path below means `profile/$PROFILE/...`. If `profile/$PROFILE/` doesn't exist, stop and tell the user to run `/switch-profile` to pick a valid one.
- Read `profile/$PROFILE/background.md` **in full** — this is the only source of truth for this profile's target roles, domain, and location scope per track. Some profiles don't use all three tracks (a track section may say "not used for this profile" — skip it, don't invent a scope for it). Read `profile/$PROFILE/cv.md` (industry/europe) and `profile/$PROFILE/cv-phd.md` (phd) if that track is in scope. If a needed CV file still contains only placeholder text, warn the user at the end but continue using `background.md` alone.
- Read `.env.local` for `SUPABASE_SERVICE_ROLE_KEY`. If it's empty, stop and tell the user to complete `SETUP.md` step 5 first.

## 1. Fetch existing tracker rows (for dedupe)
```
curl "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?select=company,role,link,track&profile=eq.$PROFILE" \
  -H "apikey: <service role key>" \
  -H "Authorization: Bearer <service role key>"
```
Filtered to `$PROFILE` — each profile's tracker is independent, so the same external role can legitimately appear in more than one person's list without counting as a duplicate. Keep `(company, role, link, track)` in memory for dedupe in step 4. Service role key bypasses RLS — local use only, never expose it in `index.html` or commit it.

## 2. Discovery subagents (one per track being run)
For each track actually in scope for `$PROFILE` (per `background.md` — skip any track that file says isn't used for this profile), launch a `general-purpose` Agent (fresh, not a fork) with a self-contained prompt containing:
- The full text of that track's section from `profile/$PROFILE/background.md` — target roles, seniority, domain, and location scope all come from there, not from any assumption baked into this command.
- Instruction: search general boards (LinkedIn Jobs, Indeed, Glassdoor) plus whichever sector-specific boards, national job boards, or company career pages are actually relevant to the domain and locations described in that section — pick boards suited to the industry and geography named there (e.g. a UK offshore-wind search and an India-based refining/asset-integrity search call for entirely different boards; use judgement, don't default to one profile's boards for another). For a `phd` track, search FindAPhD, jobs.ac.uk, and university department funding/studentship pages instead of general job boards — location is secondary to research fit there.
- If the section says something like "weight X first, then also search Y" (e.g. a home city/country weighted first with wider search beyond it), follow that ordering — surface results from the weighted-first scope but don't exclude the rest.

Each agent returns a structured list per candidate: `role, company, location, link, source_url, jd_text` (jd_text = a few sentences summarizing actual requirements, not just the search snippet). Cap each track at ~15-25 candidates. Run all requested tracks' discovery agents in parallel (single message, multiple Agent calls).

## 3. Dedupe
Drop any candidate matching an existing row from step 1 by `link`, or by `(company, role, track)` if the link differs but both match closely.

## 4. Fit-scoring
For each remaining new candidate, compare its `jd_text` against the relevant profile file(s) — `profile/$PROFILE/background.md` + `cv.md` for `industry`/`europe`, `background.md` + `cv-phd.md` for `phd`. Assign:
- `tier`: `A` (strong direct match), `B` (adjacent/plausible fit), `C` (stretch/loosely related)
- `notes`: one grounded sentence explaining *why*, referencing the profile's actual background from `background.md`'s "Why this person is a fit" section — real, specific facts (past roles, certifications, concrete project results) — not generic filler.

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
Report to the user: how many new roles were added per track, broken down by tier within each, and call out the top 2-3 standout matches overall by name with a one-line reason. Remind them the live tracker will show them next time they open/refresh the page (after signing in), under the `$PROFILE` profile card, filterable further by the track pills.
