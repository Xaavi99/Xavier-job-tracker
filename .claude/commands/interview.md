---
description: Prepare for an interview on a tracked job, then draft the follow-up email afterwards
argument-hint: <company name or job id> [after]
---

Interview support for one job in the active profile's tracker (`$PROFILE`, from `.active-profile`). Target: $ARGUMENTS. Match the job the same way as `tailor-application.md` step 1. If the arguments include `after`, skip to **After the interview**.

Use only facts from `profile/$PROFILE/cv.md`, `background.md`, the job's `applications` row (the CV and letter actually sent) and `voice-notes.md`. No em dash anywhere.

## Before the interview
Write `applications/<job folder>/interview-prep.md` with:
1. **Their top requirements** (from the JD, in its words), each with the one story from the sent CV that proves it. Tell each story outcome first in plain language (what changed, with a number if there is one), then the method, so a non-specialist on the panel can follow it.
2. **Likely hard questions** on the gaps in the requirement map, with honest answers built on the closest real evidence.
3. **When you don't know:** say so plainly ("I don't know yet, but I'll find out"), note the question, and answer it by email the same day. Keep a running list of these in the prep file.
4. **Closing question, every time:** "Is there anything about my background that gives you hesitation?" Prepare a short answer for each concern the requirement map predicts, so it can be dealt with in the room.
5. Three questions to ask them, specific to the role and company.

Set the job's `status` to `Interview` in `jobs` if it isn't already.

## After the interview
Ask the user for: interviewer names and emails, one or two specific things discussed, any question left open or answered with "I'll find out", and any hesitation they raised.

Draft a short thank-you and follow-up email, to be sent **within 24 hours**:
- Thank them, and reference something specific from the conversation.
- Answer anything left open, with the actual answer.
- If a hesitation came up, address it in one or two sentences with real evidence.
- Under 150 words, in the person's own voice.

Queue it in `automation_log` as `kind='email'`, `status='awaiting_approval'`, with the full draft in `payload`. Never send it automatically. Report the draft and the deadline (interview time plus 24 hours).
