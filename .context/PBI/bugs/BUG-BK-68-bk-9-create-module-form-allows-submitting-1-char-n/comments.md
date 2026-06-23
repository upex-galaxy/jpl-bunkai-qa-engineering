# Comments for BK-68

[View in Jira](https://jira.upexgalaxy.com/browse/BK-68)

---

### Automation for Jira - 10/6/2026, 21:18:32

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 10/6/2026, 21:18:33

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 10/6/2026, 21:34:20

## Root Cause Analysis

The Create Module form's submit gate only checked non-empty input (`name.trim().length > 0`), while the API enforces `MIN*NAME*LENGTH = 2` — so a 1-character name enabled Submit and burned a server round-trip just to receive the 422 `name*too*short`.

## Fix Applied — already fixed before this pass

This was structurally fixed on ***2026-06-08*** (two days after the report) by commit `df47918` — "fix(forms): align client min-length validation with the server":

```
- const isValid = name.trim().length > 0 && !submitting;
+ const isValid = trimmedName.length >= 2 && !submitting;
```

The Submit button is now disabled until the trimmed name reaches 2 characters, matching the server constraint — the Expected Result's "disable submit" option. No additional code change was needed in this pass; the current staging build (PRs #36/#37 era) carries the fix.

## Verification Performed

- Code verified on current `staging`: `create-module-form.tsx` gates `isValid = trimmedName.length >= 2 && !submitting` and the button binds `disabled={!isValid}`.
- Full suite green on staging HEAD (243/243).

## How to Verify

Open New Module, type exactly 1 character → Submit stays disabled (no server call, no 422). Type 2+ characters → Submit enables.

**Fix ready for QA verification.**

---

### Andrés Daniel Cumare Morales - 14/6/2026, 16:50:28

QA Bug Verification - BK-68

Environment: Staging
Result: VERIFIED - Bug fix confirmed

TEST DATA USED:

- Project: BK-9 Module Test Project (ID: ae10a3bd-574f-4caf-8076-f19a8e80f5a6)

VERIFICATION:

- Original bug scenario: No longer reproduces. Typing a 1-character name ("X") into the create-module name field no longer allows submission.
- Expected behavior: Now works correctly. The form shows the inline validation message "Name must be at least 2 characters." and disables the "Create module" submit button.
- Regression check: No issues found. Typing a 2-character name ("XY") clears the validation message, re-enables the submit button, and the create succeeds end-to-end (API returns 201).

---


_Synced from Jira by sync-jira-issues_
