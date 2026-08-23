---
description: Switch which profile folder /tailor-application and /find-jobs read from
argument-hint: "<profile name> (omit to show the current profile and list available ones)"
---

Manage which `profile/<name>/` folder is "active" — the one `/tailor-application` and `/find-jobs` read `cv.md`/`cv-grad.md`/`cv-phd.md`/`background.md`/`personal-statement.md`/`voice-notes.md` from.

Argument: $ARGUMENTS

## If no argument was given
Read `.active-profile` (if missing, the active profile is `xavier` by default). List the folders under `profile/` (excluding `_template`). Report the current active profile and the full list of available ones.

## If a profile name was given
1. Check `profile/<name>/` exists.
   - If it doesn't: list the available profiles under `profile/` (excluding `_template`) and stop — don't create anything. If they're trying to create a brand new profile, tell them to copy `profile/_template/` to `profile/<name>/` and fill in the files first, then run this command again.
2. If it exists, write the name (just the name, no trailing content) to `.active-profile`, overwriting whatever was there.
3. Confirm: "Active profile is now `<name>` — /tailor-application and /find-jobs will use profile/<name>/ from here on."
