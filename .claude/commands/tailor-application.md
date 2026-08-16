---
description: Generate a tailored CV and cover letter (or PhD personal statement) for one tracked job
argument-hint: <company name or job id>
---

Generate tailored application materials for one job from Xavier's tracker. The target job is: $ARGUMENTS (match it against company name or numeric id — if ambiguous or not found, list close matches and ask which one).

## 0. Load context
- Read `profile/background.md` and `profile/voice-notes.md` (needed for every track).
- Read `.env.local` for `SUPABASE_SERVICE_ROLE_KEY`. If empty, stop and point to `SETUP.md` step 5.

## 1. Fetch the job row
```
curl "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?select=*&or=(company.ilike.*<query>*,id.eq.<id>)" \
  -H "apikey: <service role key>" \
  -H "Authorization: Bearer <service role key>"
```
Use `jd_text`, `link`, and **`track`** from the matched row. If `jd_text` is empty (e.g. an old seed row from before this field existed), fetch the `link` and extract the actual requirements yourself before continuing.

## 2. Branch on track

### If `track` is `industry` or `europe`
Read `profile/cv.md`. If it's still just the placeholder, stop and tell Xavier to paste his CV into it first.

**CV-tailoring subagent** — `general-purpose` Agent (fresh) with a self-contained prompt containing: full `profile/cv.md`, the job's `jd_text`/role/company/location, and the instruction to restructure/re-emphasize (not fabricate) — reorder bullets, surface the most relevant experience first, mirror the JD's own terminology where it's an honest match (ATS alignment), stay truthful to the source content. Output: complete tailored CV in Markdown.

**Cover-letter subagent** — second `general-purpose` Agent (fresh) with: `profile/background.md`, `profile/voice-notes.md` (match this register — concrete hook, honest about what he learned not just achieved, real numbers, no stock phrases), the job's `jd_text`/role/company. Instruction: concise, specific cover letter referencing 1-2 concrete things about the company/role, connected to Xavier's actual RBI/Asset Integrity background and CTMC/MPC dissertation (49% OPEX reduction, 6% availability improvement, 15MW spar-type case study) where genuinely relevant. No filler. Output: complete cover letter in Markdown.

Run both agents in parallel. Save to `applications/<slug>/cv.md` and `applications/<slug>/cover-letter.md`.

### If `track` is `phd`
Read `profile/cv-phd.md` and `profile/personal-statement.md`. If `cv-phd.md` is still placeholder-only, stop and tell Xavier to fill it in.

**Research CV subagent** — `general-purpose` Agent (fresh) with: full `profile/cv-phd.md`, the programme's `jd_text`/institution/supervisor names if known. Instruction: restructure to foreground the research fit most relevant to this specific programme's stated focus, keep the research-interest framing (not a job-market CV), stay truthful to source content. Output: complete Markdown CV.

**Personal statement / research-fit subagent** — second `general-purpose` Agent (fresh) with: `profile/personal-statement.md`, `profile/background.md`'s dissertation section (real numbers + the self-identified static-accessibility-windows limitation), `profile/voice-notes.md`, and the programme's `jd_text`/institution/named supervisors. Instruction: mirror the existing throughline — dissertation's real limitation → this specific programme's stated research gap — don't reuse the Strathclyde text verbatim, write a new one grounded in what THIS programme is actually researching. Output: complete Markdown personal statement / research-fit letter.

Run both agents in parallel. Save to `applications/<slug>/cv.md` and `applications/<slug>/personal-statement.md` (not `cover-letter.md` — PhD applications use a personal statement, not a cover letter).

## 3. Update the tracker row
For `industry`/`europe`:
```
curl -X PATCH "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?id=eq.<id>" \
  -H "apikey: <service role key>" -H "Authorization: Bearer <service role key>" \
  -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  -d '{"cv_path":"applications/<slug>/cv.md","cover_letter_path":"applications/<slug>/cover-letter.md"}'
```
For `phd`, same but `"cover_letter_path":"applications/<slug>/personal-statement.md"`.

## 4. Report
Tell Xavier where the files landed and give a 2-3 sentence summary of the angle each document took. Remind him these are drafts for review — nothing gets submitted automatically.
