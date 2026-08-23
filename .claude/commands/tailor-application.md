---
description: Generate a tailored, ATS-scored CV and cover letter (or PhD personal statement) for one tracked job
argument-hint: <company name or job id>
---

Generate tailored application materials for one job from the active profile's tracker (`$PROFILE`, from `.active-profile`). The target job is: $ARGUMENTS (match it against company name or numeric id — if ambiguous or not found, list close matches and ask which one).

**Non-negotiable constraints, apply to every subagent prompt below:**
- **No fabrication.** Restructure, reorder, and re-emphasize real content from `profile/$PROFILE/cv.md`/`cv-phd.md` and `background.md` only. Never invent a skill, tool, certification, metric, or experience. If the JD wants something the profile doesn't have, omit it or bridge from a genuinely adjacent real skill — never claim it outright. **When bridging, lead with the transferable evidence, not the gap.** Open on the closest real project/tool/result from `background.md` or the CV that maps to what the JD wants, and name the specific thing they haven't done afterward, briefly, if at all — never open a sentence with "I haven't / I don't have / I'm not going to pretend." The gap can still be acknowledged; it just isn't the first thing a reader or ATS keyword pass hits.
- **Sound like them, not like AI.** Pass `profile/$PROFILE/voice-notes.md` to every drafting subagent in full — it has their real register plus a hard list of AI-tell phrasing to avoid (em dashes as connectors, "furthermore/moreover," "leverage/utilize," passion-as-adjective, rule-of-three padding, repeated sentence patterns). Enforce it, don't just mention it.
- **Genuine interest in the target domain, shown not claimed.** No "I am passionate about X." Show it through the specific real reasons in `profile/$PROFILE/background.md`'s "Why this person is a fit" section (and `personal-statement.md` for PhD tracks) — each profile has its own concrete motivations/story there; use theirs, not a generic one.
- **ATS-parseable.** Standard section headers (Profile/Experience/Education/Skills, not creative labels), no tables/columns/graphics, dates in a consistent format, acronyms spelled out at least once (e.g. "Risk-Based Inspection (RBI)"), JD keywords mirrored verbatim wherever they honestly match the profile's real experience.
- **Profile section stays short.** 3-4 sentences, roughly 50-70 words — a recruiter skims a CV in seconds, and a dense paragraph doesn't get read. Every sentence must carry a real fact, skill, or JD keyword. Cut connective/narrative scene-setting ("that work covered...", "that foundation already includes..."), not the substance — if a JD term (e.g. a specific tool or technique the JD asks for that the profile is honestly bridging from adjacent experience) appears nowhere else in the CV, keep it in the trimmed Profile rather than cutting it for length; check before finalizing that no JD keyword present in the untrimmed draft has been dropped from the CV entirely. Lead with the strongest direct match to the JD, not chronological scene-setting.

## 0. Load context
- Read `.active-profile` to get `$PROFILE` (if the file is missing, `$PROFILE` is `xavier`). Every `profile/...` path below means `profile/$PROFILE/...`. If `profile/$PROFILE/` doesn't exist, stop and tell the user to run `/switch-profile` to pick a valid one.
- Read `profile/$PROFILE/background.md` and `profile/$PROFILE/voice-notes.md` (needed for every track).
- Read `.env.local` for `SUPABASE_SERVICE_ROLE_KEY`. If empty, stop and point to `SETUP.md` step 5.

## 1. Fetch the job row
```
curl "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?select=*&profile=eq.$PROFILE&or=(company.ilike.*<query>*,id.eq.<id>)" \
  -H "apikey: <service role key>" \
  -H "Authorization: Bearer <service role key>"
```
Scoped to `$PROFILE` — only match against the active profile's own tracked jobs. Use `jd_text`, `link`, and **`track`** from the matched row. If `jd_text` is empty (e.g. an old seed row from before this field existed), fetch the `link` and extract the actual requirements yourself before continuing.

## 2. Generate CV + cover letter/personal statement (parallel)

### If `track` is `industry` or `europe`
**First check if this is a graduate scheme**: if the role title or `jd_text` says "graduate", "grad scheme", "graduate programme", "graduate trainee", or "early careers", use `profile/$PROFILE/cv-grad.md` as the source CV instead of `profile/$PROFILE/cv.md` — it leads with academic credentials rather than industry experience, which is the stronger opener for schemes that screen on academic performance first. Otherwise use `profile/$PROFILE/cv.md` as normal, which leads with direct professional experience. Note the choice in the final report.

Read the chosen CV source file. If it's still just the placeholder, stop and tell the user to fill it in first.

**CV-tailoring subagent** — `general-purpose` Agent (fresh) with a self-contained prompt containing: the full chosen CV source file, full `profile/$PROFILE/voice-notes.md`, the job's `jd_text`/role/company/location, and the constraints above (no fabrication, ATS-parseable, sound like them). Instruction: reorder bullets to surface the most relevant experience first, mirror the JD's own terminology only where it's an honest match. Output: complete tailored CV in Markdown.

**Cover-letter subagent** — second `general-purpose` Agent (fresh) with: `profile/$PROFILE/background.md`, full `profile/$PROFILE/voice-notes.md`, the job's `jd_text`/role/company, and the constraints above. Instruction: concise, specific cover letter referencing 1-2 concrete things about the company/role, connected to the profile's actual background as described in `background.md`'s "Why this person is a fit" section (real, concrete facts and numbers from there — not generic claims) where genuinely relevant. No filler. Output: complete cover letter in Markdown.

### If `track` is `phd`
Read `profile/$PROFILE/cv-phd.md` and `profile/$PROFILE/personal-statement.md`. If `cv-phd.md` is still placeholder-only, stop and tell the user to fill it in.

**Research CV subagent** — `general-purpose` Agent (fresh) with: full `profile/$PROFILE/cv-phd.md`, full `profile/$PROFILE/voice-notes.md`, the programme's `jd_text`/institution/supervisor names if known, and the constraints above. Instruction: restructure to foreground the research fit most relevant to this specific programme's stated focus, keep the research-interest framing (not a job-market CV). Output: complete Markdown CV.

**Personal statement subagent** — second `general-purpose` Agent (fresh) with: `profile/$PROFILE/personal-statement.md`, `profile/$PROFILE/background.md`'s dissertation/major-project section, full `profile/$PROFILE/voice-notes.md`, the programme's `jd_text`/institution/named supervisors, and the constraints above. Instruction: mirror the existing throughline in `personal-statement.md` — a real limitation or open question from past work → this specific programme's stated research gap — write a new one grounded in what THIS programme actually researches, don't reuse the source `personal-statement.md` text verbatim for a different programme.

Run both subagents (CV + cover letter/personal statement) in parallel — single message, two Agent calls.

## 3. ATS scoring
Once the tailored CV text is back, launch a third `general-purpose` Agent (fresh) with: the tailored CV text, the job's `jd_text`. Instruction: act as an ATS keyword/structure matcher. Score 0-100 based on (a) how many of the JD's explicit required skills/qualifications appear verbatim or near-verbatim in the CV, (b) standard section structure and parseable formatting, (c) no missing critical hard requirements (e.g. a specific certification or years-of-experience the JD treats as mandatory). Return a JSON object: `{"score": <int>, "notes": "<2-3 sentences: what matched well, what's missing or weak, one concrete fix if score < 80>"}`. This is separate from a "how good is this candidate" judgment — it's specifically about ATS-style keyword/structure match.

## 4. Save output

**Local files** (for easy copy/paste or PDF conversion): write to `applications/$PROFILE/<company-slug>-<role-slug>/cv.md` and `.../cover-letter.md` (or `.../personal-statement.md` for `phd`). Create the directory if needed.

**Database row** — insert into the `applications` table:
```
curl -X POST "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/applications" \
  -H "apikey: <service role key>" -H "Authorization: Bearer <service role key>" \
  -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  -d '{"job_id":<id>,"track":"<track>","profile":"$PROFILE","cv_content":"<full tailored CV markdown>","cover_letter_content":"<full cover letter/personal statement markdown>","ats_score":<int>,"ats_notes":"<notes>"}'
```
Escape the markdown content as valid JSON strings (newlines as `\n`, quotes escaped). Don't set `id` — it auto-increments.

**Tracker row pointers** — still update `jobs` so the UI's materials indicator works:
```
curl -X PATCH "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?id=eq.<id>" \
  -H "apikey: <service role key>" -H "Authorization: Bearer <service role key>" \
  -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  -d '{"cv_path":"applications/$PROFILE/<slug>/cv.md","cover_letter_path":"applications/$PROFILE/<slug>/cover-letter.md"}'
```

## 5. Report
Report to the user: the ATS score, a one-line reason for it, where the local files landed, and a 2-3 sentence summary of the angle each document took. Remind them these are drafts for review — nothing gets submitted automatically. If the ATS score is below ~70, flag the specific gap from `ats_notes` so they know what to look at before applying.
