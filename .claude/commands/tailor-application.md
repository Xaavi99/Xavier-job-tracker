---
description: Generate a tailored, ATS-scored CV and cover letter (or PhD personal statement) for one tracked job
argument-hint: <company name or job id>
---

Generate tailored application materials for one job from Xavier's tracker. The target job is: $ARGUMENTS (match it against company name or numeric id — if ambiguous or not found, list close matches and ask which one).

**Non-negotiable constraints, apply to every subagent prompt below:**
- **No fabrication.** Restructure, reorder, and re-emphasize real content from `cv.md`/`cv-phd.md` and `background.md` only. Never invent a skill, tool, certification, metric, or experience. If the JD wants something Xavier doesn't have, omit it or bridge from a genuinely adjacent real skill — never claim it outright.
- **Sound like him, not like AI.** Pass `profile/voice-notes.md` to every drafting subagent in full — it has his real register plus a hard list of AI-tell phrasing to avoid (em dashes as connectors, "furthermore/moreover," "leverage/utilize," passion-as-adjective, rule-of-three padding, repeated sentence patterns). Enforce it, don't just mention it.
- **Genuine offshore wind interest, shown not claimed.** No "I am passionate about offshore wind." Show it through the specific real reasons in `background.md`/`personal-statement.md` (the digital-twin realization, the Fugro connection, wanting research tested against real turbines).
- **ATS-parseable.** Standard section headers (Profile/Experience/Education/Skills, not creative labels), no tables/columns/graphics, dates in a consistent format, acronyms spelled out at least once (e.g. "Risk-Based Inspection (RBI)"), JD keywords mirrored verbatim wherever they honestly match Xavier's real experience.

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

## 2. Generate CV + cover letter/personal statement (parallel)

### If `track` is `industry` or `europe`
Read `profile/cv.md`. If it's still just the placeholder, stop and tell Xavier to paste his CV into it first.

**CV-tailoring subagent** — `general-purpose` Agent (fresh) with a self-contained prompt containing: full `profile/cv.md`, full `profile/voice-notes.md`, the job's `jd_text`/role/company/location, and the constraints above (no fabrication, ATS-parseable, sound like him). Instruction: reorder bullets to surface the most relevant experience first, mirror the JD's own terminology only where it's an honest match. Output: complete tailored CV in Markdown.

**Cover-letter subagent** — second `general-purpose` Agent (fresh) with: `profile/background.md`, full `profile/voice-notes.md`, the job's `jd_text`/role/company, and the constraints above. Instruction: concise, specific cover letter referencing 1-2 concrete things about the company/role, connected to Xavier's actual RBI/Asset Integrity background and CTMC/MPC dissertation (49% OPEX reduction, 6% availability improvement, 15MW spar-type case study) where genuinely relevant. No filler. Output: complete cover letter in Markdown.

### If `track` is `phd`
Read `profile/cv-phd.md` and `profile/personal-statement.md`. If `cv-phd.md` is still placeholder-only, stop and tell Xavier to fill it in.

**Research CV subagent** — `general-purpose` Agent (fresh) with: full `profile/cv-phd.md`, full `profile/voice-notes.md`, the programme's `jd_text`/institution/supervisor names if known, and the constraints above. Instruction: restructure to foreground the research fit most relevant to this specific programme's stated focus, keep the research-interest framing (not a job-market CV). Output: complete Markdown CV.

**Personal statement subagent** — second `general-purpose` Agent (fresh) with: `profile/personal-statement.md`, `profile/background.md`'s dissertation section, full `profile/voice-notes.md`, the programme's `jd_text`/institution/named supervisors, and the constraints above. Instruction: mirror the existing throughline — dissertation's real limitation → this specific programme's stated research gap — write a new one grounded in what THIS programme actually researches, don't reuse the Strathclyde text verbatim. Output: complete Markdown personal statement.

Run both subagents (CV + cover letter/personal statement) in parallel — single message, two Agent calls.

## 3. ATS scoring
Once the tailored CV text is back, launch a third `general-purpose` Agent (fresh) with: the tailored CV text, the job's `jd_text`. Instruction: act as an ATS keyword/structure matcher. Score 0-100 based on (a) how many of the JD's explicit required skills/qualifications appear verbatim or near-verbatim in the CV, (b) standard section structure and parseable formatting, (c) no missing critical hard requirements (e.g. a specific certification or years-of-experience the JD treats as mandatory). Return a JSON object: `{"score": <int>, "notes": "<2-3 sentences: what matched well, what's missing or weak, one concrete fix if score < 80>"}`. This is separate from a "how good is this candidate" judgment — it's specifically about ATS-style keyword/structure match.

## 4. Save output

**Local files** (for easy copy/paste or PDF conversion): write to `applications/<company-slug>-<role-slug>/cv.md` and `.../cover-letter.md` (or `.../personal-statement.md` for `phd`). Create the directory if needed.

**Database row** — insert into the `applications` table:
```
curl -X POST "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/applications" \
  -H "apikey: <service role key>" -H "Authorization: Bearer <service role key>" \
  -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  -d '{"job_id":<id>,"track":"<track>","cv_content":"<full tailored CV markdown>","cover_letter_content":"<full cover letter/personal statement markdown>","ats_score":<int>,"ats_notes":"<notes>"}'
```
Escape the markdown content as valid JSON strings (newlines as `\n`, quotes escaped). Don't set `id` — it auto-increments.

**Tracker row pointers** — still update `jobs` so the UI's materials indicator works:
```
curl -X PATCH "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?id=eq.<id>" \
  -H "apikey: <service role key>" -H "Authorization: Bearer <service role key>" \
  -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  -d '{"cv_path":"applications/<slug>/cv.md","cover_letter_path":"applications/<slug>/cover-letter.md"}'
```

## 5. Report
Tell Xavier: the ATS score, a one-line reason for it, where the local files landed, and a 2-3 sentence summary of the angle each document took. Remind him these are drafts for review — nothing gets submitted automatically. If the ATS score is below ~70, flag the specific gap from `ats_notes` so he knows what to look at before applying.
