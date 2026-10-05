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
- **Placeholders, not omissions or guesses.** When the JD needs a fact only the user can supply or confirm (licence, certificate expiry, availability date, a gap explanation, the official name of a course), write a highlighted placeholder instead of dropping it or guessing: `==[CONFIRM: ...]==` (requirement the CV doesn't state and claiming it would be risky), `==[DATE: ...]==` (a needed date or expiry), `==[DETAIL: ...]==` (a real fact missing the specifics a recruiter expects), `==[GAP: Mon YYYY to Mon YYYY, ...]==` (a gap of 3+ months between roles), `==[CHECK: ...]==` (a term or certificate name that can't be verified). The sentence must still read correctly once the placeholder is resolved. Use the same placeholder text in the CV and cover letter, so one answer fixes both. A placeholder is never a way to sneak in a claim: if the profile clearly lacks something (e.g. CSWIP 3.4U, Dutch), it stays an honest gap, not a CONFIRM.
- **UK conventions.** No photo, declaration, signature, date of birth or "References available on request" line. Named academic referees are fine as a plain line ("Referees: Prof. Jin Wang and Dr Alireza E. Majd, Liverpool John Moores University"). Right to work is stated in the header in words, not implied. Present tense for the current role only, past tense for everything else. Date ranges keep the existing format (en dash allowed per voice-notes.md), consistently.
- **Profile section stays short.** 3-4 sentences, roughly 50-70 words — a recruiter skims a CV in seconds, and a dense paragraph doesn't get read. Every sentence must carry a real fact, skill, or JD keyword. Cut connective/narrative scene-setting ("that work covered...", "that foundation already includes..."), not the substance — if a JD term (e.g. a specific tool or technique the JD asks for that the profile is honestly bridging from adjacent experience) appears nowhere else in the CV, keep it in the trimmed Profile rather than cutting it for length; check before finalizing that no JD keyword present in the untrimmed draft has been dropped from the CV entirely. Lead with the strongest direct match to the JD, not chronological scene-setting.

## 0. Load context
- Read `.active-profile` to get `$PROFILE` (if the file is missing, `$PROFILE` is `uk`). Every `profile/...` path below means `profile/$PROFILE/...`. If `profile/$PROFILE/` doesn't exist, stop and tell the user to run `/switch-profile` to pick a valid one.
- Read `profile/$PROFILE/background.md` and `profile/$PROFILE/voice-notes.md` (needed for every track).
- Read `.env.local` for `SUPABASE_SERVICE_ROLE_KEY`. If empty, stop and point to `SETUP.md` step 5.

## 1. Fetch the job row
```
curl "https://nthonxtycsyplkmpwjrh.supabase.co/rest/v1/jobs?select=*&profile=eq.$PROFILE&or=(company.ilike.*<query>*,id.eq.<id>)" \
  -H "apikey: <service role key>" \
  -H "Authorization: Bearer <service role key>"
```
Scoped to `$PROFILE` — only match against the active profile's own tracked jobs. Use `jd_text`, `link`, and **`track`** from the matched row. If `jd_text` is empty (e.g. an old seed row from before this field existed), fetch the `link` and extract the actual requirements yourself before continuing.

## 1b. Requirement map (before any drafting)
Parse the JD into four lists:
- **Knockouts:** anything phrased as must / minimum / required / valid / essential, right to work, licence, location or commute, years of experience, named certificates. These usually become screening questions.
- **Preferred:** preferred / ideally / desirable / a plus.
- **Keywords:** the exact nouns and phrases the JD uses for duties and skills, in the JD's own spelling.
- **Motivators:** what the employer is selling (route to chartership, offshore rotation, graduate rotations). Used in the profile and cover letter.

Then build a map, one row per knockout and preferred item:

| JD requirement | Type | Evidence in profile | Action |
|---|---|---|---|
| e.g. MSc in marine/mechanical | Knockout | MSc Marine and Offshore (Distinction) | State in headline + Key Qualifications |
| e.g. Full UK driving licence | Knockout | Not stated | `==[CONFIRM: Full UK driving licence]==` |
| e.g. CSWIP 3.4U | Knockout | Not held | Honest gap: name the closest real evidence, flag in report |

**Every knockout must end as evidenced, a placeholder, or an explicitly flagged honest gap. Never leave one unaddressed.** Also run a gap check on the source CV's dates: any gap of 3+ months between roles or study gets a `==[GAP: ...]==` unless `background.md` already explains it. Pass the map to every drafting subagent and the ATS scorer.

## 2. Generate CV + cover letter/personal statement (parallel)

### If `track` is `industry` or `europe`
**First check if this is a graduate scheme**: if the role title or `jd_text` says "graduate", "grad scheme", "graduate programme", "graduate trainee", or "early careers", use `profile/$PROFILE/cv-grad.md` as the source CV instead of `profile/$PROFILE/cv.md` — it leads with academic credentials rather than industry experience, which is the stronger opener for schemes that screen on academic performance first. Otherwise use `profile/$PROFILE/cv.md` as normal, which leads with direct professional experience. Note the choice in the final report.

Read the chosen CV source file. If it's still just the placeholder, stop and tell the user to fill it in first.

**CV-tailoring subagent** — `general-purpose` Agent (fresh) with a self-contained prompt containing: the full chosen CV source file, full `profile/$PROFILE/voice-notes.md`, the job's `jd_text`/role/company/location, and the constraints above (no fabrication, ATS-parseable, sound like them). Instruction: reorder bullets to surface the most relevant experience first, mirror the JD's own terminology only where it's an honest match. Also: (1) the bold headline line under the name uses the JD's job title exactly; (2) the header line states right to work and any knockout items such as location or relocation; (3) if the JD has 2+ knockouts, add a short **Key Qualifications** section directly after the Profile that answers them point by point in the JD's order, using evidence or the map's placeholders; (4) work each JD keyword in naturally at least once, never repeated artificially and never as hidden text; (5) order the skills so the JD's top priorities come first; (6) relabel a role only where the source duties support it. Output: complete tailored CV in Markdown, plus the list of placeholders used.

**Cover-letter subagent** — second `general-purpose` Agent (fresh) with: `profile/$PROFILE/background.md`, full `profile/$PROFILE/voice-notes.md`, the job's `jd_text`/role/company, and the constraints above. Instruction: one page, 5 to 6 short paragraphs: opening (role, headline qualification, single strongest match), recent experience mirroring the JD's duties, background in one paragraph, motivation tied to one JD motivator, practicalities (knockouts such as location, visa, licence, using the same placeholders as the CV, and weaknesses addressed directly), close. Concise and specific, referencing 1-2 concrete things about the company/role, connected to the profile's actual background as described in `background.md`'s "Why this person is a fit" section (real, concrete facts and numbers from there — not generic claims) where genuinely relevant. No filler. Output: complete cover letter in Markdown.

### If `track` is `phd`
Read `profile/$PROFILE/cv-phd.md` and `profile/$PROFILE/personal-statement.md`. If `cv-phd.md` is still placeholder-only, stop and tell the user to fill it in.

**Research CV subagent** — `general-purpose` Agent (fresh) with: full `profile/$PROFILE/cv-phd.md`, full `profile/$PROFILE/voice-notes.md`, the programme's `jd_text`/institution/supervisor names if known, and the constraints above. Instruction: restructure to foreground the research fit most relevant to this specific programme's stated focus, keep the research-interest framing (not a job-market CV). Output: complete Markdown CV.

**Personal statement subagent** — second `general-purpose` Agent (fresh) with: `profile/$PROFILE/personal-statement.md`, `profile/$PROFILE/background.md`'s dissertation/major-project section, full `profile/$PROFILE/voice-notes.md`, the programme's `jd_text`/institution/named supervisors, and the constraints above. Instruction: mirror the existing throughline in `personal-statement.md` — a real limitation or open question from past work → this specific programme's stated research gap — write a new one grounded in what THIS programme actually researches, don't reuse the source `personal-statement.md` text verbatim for a different programme.

Run both subagents (CV + cover letter/personal statement) in parallel — single message, two Agent calls.

## 3. ATS scoring
Once the tailored CV text is back, launch a third `general-purpose` Agent (fresh) with: the tailored CV text, the job's `jd_text`. Instruction: act as an ATS keyword/structure matcher. Score 0-100 based on (a) how many of the JD's explicit required skills/qualifications appear verbatim or near-verbatim in the CV, (b) standard section structure and parseable formatting, (c) no missing critical hard requirements (e.g. a specific certification or years-of-experience the JD treats as mandatory). Use the step 1b map for (c): weights are keywords 60, structure 20, mandatory 20; each knockout that is evidenced costs 0, each one sitting on an unresolved placeholder costs 5 (it counts as partial until the user confirms it), and each honest gap costs 10. Return a JSON object: `{"score": <int>, "notes": "<2-3 sentences: what matched well, what's missing or weak, one concrete fix if score < 80>"}`. This is separate from a "how good is this candidate" judgment — it's specifically about ATS-style keyword/structure match.

## 4. Optimize if score is low
If `score` < 80, run one revision pass before saving anything — don't just pass the low score through to the report.

**CV-revision subagent** — `general-purpose` Agent (fresh) with: the tailored CV text, the job's `jd_text`, the ATS `notes` from step 3, the original chosen CV source file (`cv.md`/`cv-grad.md`/`cv-phd.md`), and the constraints from the top of this file (no fabrication, ATS-parseable, sound like them, Profile section length limit). Instruction: address each gap named in `notes` **only by surfacing real, already-true content from the source CV/`background.md` that step 2 left out or under-emphasized** — pull in a genuinely matching skill/tool/metric that exists but didn't make it into the tailored draft, mirror more of the JD's exact phrasing where it's an honest match, fix structural issues (section headers, formatting) the notes called out. Never invent anything to close a gap that's real (no matching experience exists) — if a required item genuinely isn't in the profile, leave it as a gap. Output: complete revised CV in Markdown.

Then re-run the same ATS-scoring subagent from step 3 on the revised CV to get an updated `{"score", "notes"}`. Run this optimize-and-rescore cycle **at most once** — take whatever the second score is (even if still < 80, since further gaps at that point are likely genuine and not fabricable) and use the revised CV as the final output going forward.

## 5. Save output

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

**Draft files:** build a draft with `python ../tools/build_cv.py <cv.md> <out.docx>` and `python ../tools/build_letter.py <cover-letter.md> <out.docx> "<recipient lines>"`, then convert to PDF with LibreOffice (`"C:/Program Files/LibreOffice/program/soffice.exe" --headless --convert-to pdf`). In drafts, placeholders render with a yellow highlight and the builder prints them. File names: `Xavier_Sojan_CV_<Company>_<Role>.docx/.pdf` and `Xavier_Sojan_Cover_Letter_<Company>.docx/.pdf` in the job folder.

## 6. Report
Report to the user: the requirement map (knockouts only, one line each: evidenced / placeholder / honest gap), then an **Items to confirm** list, one numbered line per placeholder with its tag and every place it appears, e.g. `1. [CONFIRM] Full UK driving licence (CV header, Key Qualifications, cover letter para 5)`. Then the ATS score (and, if step 4 ran, the before/after score so the improvement is visible), a one-line reason for it, where the local files landed, and a 2-3 sentence summary of the angle each document took. Remind them these are drafts for review — nothing gets submitted automatically. If the final ATS score is still below ~70, flag the specific gap from `ats_notes` so they know what to look at before applying — at this point it's a genuine gap in the profile, not something the tailoring pass could fix without fabricating.

## 7. Resolve placeholders, then final build
When the user answers the Items to confirm list:
- **Confirmed with a value:** replace the placeholder with the value (in the CV and the cover letter).
- **Confirmed true:** remove the `==[TAG: ` / `]==` markers and keep the wording.
- **False:** delete the whole line or sentence; never soften it into a weaker claim. If it was a knockout, warn the user that the application may be screened out.
- **Unknown:** leave it, keep it on the list.

Update the markdown files and the `applications` row, re-run the ATS score if any knockout changed, and repeat until the list is empty. A draft with placeholders may be reviewed but **never submitted or sent**: `/send-approved` and the daily sweep must refuse any CV or letter that still contains `==[`.

**Final build:** `python ../tools/build_cv.py <cv.md> <out.docx> --final` and `python ../tools/build_letter.py <cover-letter.md> <out.docx> "<recipient>" --final`. Both refuse to build while any placeholder remains. Then convert to PDF and check:
- [ ] Zero placeholders, zero highlighted text, zero em dashes.
- [ ] Every fact traces to `cv.md` / `background.md` or a user confirmation.
- [ ] Every knockout is evidenced, or knowingly left as an honest gap the user has seen.
- [ ] Each JD keyword that honestly applies appears at least once.
- [ ] Present tense for the current role only; dates consistent; any 3+ month gaps explained or deliberately left.
- [ ] Single column, contact details in the body, standard headings.
- [ ] CV at most 2 pages, cover letter 1 page, no orphaned heading or a role split awkwardly across pages.
- [ ] File names follow the convention. Upload format: .docx for corporate ATS portals (Workday, Oracle, SuccessFactors, Taleo) unless they ask for PDF; PDF for email, LinkedIn Easy Apply and university portals.

Finish with a short note on anything the user could still strengthen.
