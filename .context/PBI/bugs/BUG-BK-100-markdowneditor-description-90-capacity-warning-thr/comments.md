# Comments for BK-100

[View in Jira](https://jira.upexgalaxy.com/browse/BK-100)

---

### Automation for Jira - 10/6/2026, 12:53:25

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 10/6/2026, 12:54:29

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 10/6/2026, 13:46:07

## 🔧 Bug Fix Documentation

### Root Cause Analysis

***Category******:*** Code Error
***Location******:*** `components/markdown/markdown-editor.tsx` (threshold math, not missing UI)

The 90% warning state (amber `text-signal-blocked` on the size counter) ***was already implemented**** — it never fired because the cap used `50 ** 1024` (51,200 bytes, KiB), putting the 90% threshold at 46,080 bytes; the 45,500-byte probe stayed below it. Same KiB/KB unit mismatch as ***BK-84's sibling BK-99*** — fixed together. The minor display discrepancy (counter divided by 1024 but labelled "KB") is also corrected.

### Fix Applied

***PR******:*** https://github.com/upex-galaxy/upex-bunkai-tms/pull/33 (merged with BK-99, commit `e61cf15`, deployed to staging)
***Fix Type******:*** Bugfix

- Editor `DEFAULT*MAX*BYTES` (and the shared server constants) → ***50,000 bytes**** ⇒ 90% threshold now ****45,000 bytes***
- Counter displays true decimal KB (`bytes / 1000`) — 45,500 bytes now reads ***45.5 KB***, not 44.4

Post-fix: neutral < 45,000 · ***amber warning 45,001–50,000*** · red + inline error + disabled submit > 50,000.

### Verification Performed

- [x] Threshold math: 45,500 > 45,000 ⇒ `nearCap` true ⇒ `text-signal-blocked` class applied (covered by review; state machine unchanged, only the constant moved)
- [x] `bun test` 184/184 · `tsc` clean · `eslint` clean
- [x] Server-side cap smoke green on staging (see BK-99 comment for the 422 evidence)

### How to Verify

1. Open the user-story form on staging, inject 45,500 chars (QA's JS snippet)
2. ***Expected******:**** counter shows ****45.5 KB*** in amber (`text-signal-blocked`), submit still enabled
3. Push past 50,000 → counter red, inline error, submit disabled

---

**Fix ready for QA verification.**

---

### Facu Barea - 10/6/2026, 20:14:22

## :test_tube: Acceptance Test Plan (ATP) — BK-100

***Type******:**** Bug Retest | ****Priority******:**** Medium | ****Environment******:**** Staging | ****Date******:*** 2026-06-10

---

### Bug Summary

The 90% capacity warning threshold was not implemented in the Markdown editor description field. When a user entered 45,500 bytes (>90% of the 50 KB limit), the size counter remained neutral color (`text-fg-4`) with no warning. AC6 requires a visible warning indicator before reaching the hard limit.

### Veto Check

No veto — missing functional AC6 behavior (not pure CSS/docs/config). Retesting REQUIRED.

### Risk Score

 — UX degradation only. No data loss, no auth impact, no data integrity concern.

---

### Test Approach

| Layer | What to verify | Method |
| --- | --- | --- |
| UI — warning state | Counter shows warning indicator at 90% | Browser JS injection |
| UI — normal state | Counter remains neutral below 90% | Browser JS injection |
| UI — blocked state | Counter shows blocked/fail state above 100% | Browser JS injection |
| UI — state distinct | Warning state is visually distinct from blocked state | Visual + class check |

---

### Test Cases

***BK-100 TC-01 — Warning at >90%******:****** counter shows warning state***

- Inject 45,500 chars (>90% of 50KB = 45,000 bytes)
- Expected: counter shows a warning indicator (color change, badge, or tooltip)
- Expected: counter class is NOT `text-fg-4` (neutral)
- Expected: submit button remains ENABLED

***BK-100 TC-02 — Normal below 90%******:****** counter stays neutral***

- Inject 44,000 chars (~88% of 50KB)
- Expected: counter in neutral state, no warning

***BK-100 TC-03 — Blocked above 100%******:****** counter shows blocked/fail state***

- Inject 51,000 chars (>50KB)
- Expected: counter shows `text-signal-fail` or blocked class
- Expected: submit button disabled (verified in BK-99 — regression check)

***BK-100 TC-04 — State distinctness******:****** warning ≠ blocked***

- Compare counter CSS class at 45,500 chars vs 51,000 chars
- Expected: different classes (warning state is NOT the same as blocked state)

---

### Test Data

```js
// 90%+ (warning zone): 45,500 chars
const el = document.querySelector('textarea[placeholder="Describe the story in Markdown."]');
const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
setter.call(el, 'A'.repeat(45500));
el.dispatchEvent(new Event('input', { bubbles: true }));
```

### Pass / Fail Criteria

> ***SUCCESS:*** PASS — counter shows distinct warning state at 45,500 chars; neutral below 90%; blocked above 50KB.

> ***ERROR:*** FAIL — counter stays neutral at 45,500 chars (no visual warning implemented).

---

### Facu Barea - 10/6/2026, 20:17:01

## :test_tube: Acceptance Test Results (ATR) — BK-100

***Type******:**** Bug Retest | ****Verdict******:****  | ****Environment******:**** Staging | ****Date******:*** 2026-06-10

---

### Summary

All 4 test cases passed. The 90% capacity warning threshold is now implemented. The counter transitions through three distinct visual states as the description grows, giving users early warning before reaching the hard limit.

---

### Counter State Machine (verified)

| Range | Bytes | Counter class | Button | Expected |
| --- | --- | --- | --- | --- |
| Normal | ≤ 45,000 | `text-fg-4` (neutral) | enabled | :white*check*mark: Correct |
| Warning | 45,001 – 50,000 | `text-signal-blocked` (amber) | enabled | :white*check*mark: Correct |
| Blocked | > 50,000 | `text-signal-fail` (red) | disabled | :white*check*mark: Correct |

---

### Test Execution Results

| TC | Description | Result | Notes |
| --- | --- | --- | --- |
| TC-01 | Warning at >90%: counter shows warning state | :white*check*mark: PASS | `text-signal-blocked` at 45,500 chars · button remains enabled |
| TC-02 | Normal below 90%: counter stays neutral | :white*check*mark: PASS | `text-fg-4` at 44,000 chars |
| TC-03 | Blocked above 100%: counter shows fail state | :white*check*mark: PASS | `text-signal-fail` at 51,000 chars · button disabled (regression: BK-99 fix still holding) |
| TC-04 | State distinctness: warning ≠ blocked | :white*check*mark: PASS | `text-signal-blocked` (warning) is visually and programmatically distinct from `text-signal-fail` (blocked) |

---

### Evidence

- `bk100-tc01-warning-state-45500.png` — counter at 45.5 KB in amber warning state, submit button still enabled

---

### Observations

> ***INFO:*** The warning threshold triggers at >45,000 bytes (i.e., 45,001+). At exactly 45,000 bytes the counter remains neutral. The threshold is 90% of 50,000 bytes — this is correct per AC6.

> ***INFO:*** The counter display at 45,001 bytes rounds down to "45.0 KB" (45001/1000 = 45.001 → 45.0 displayed). This is a minor cosmetic rounding artifact in the display, not a functional issue.

> ***INFO:*** KiB vs KB: the counter now uses 1000 as the divisor (showing "45.5 KB" at 45,500 bytes). This is an improvement from the original bug report which noted the counter was using 1024.

---

### Verdict

 — Bug is ***fixed***. 90% warning threshold implemented with a distinct amber state. Ready to close.

With BK-99 also closed, BK-16 can now be unblocked.

---

### Facu Barea - 10/6/2026, 20:17:14

## :white*check*mark: QA Sign-Off — BK-100 VERIFIED FIXED

***Tester******:**** Facu Barea | ****Date******:**** 2026-06-10 | ****Environment******:*** Staging

The fix for BK-100 has been verified. The 90% capacity warning threshold is now implemented with a distinct visual state:

- ***Normal*** (≤45 KB): neutral counter, no warning
- ***Warning*** (45–50 KB): amber counter `text-signal-blocked`, submit enabled
- ***Blocked*** (>50 KB): red counter `text-signal-fail`, submit disabled

All 4 test cases passed. The warning gives users early feedback before hitting the hard limit, exactly as AC6 required. Closing this bug. :closed*lock*with_key:

Both BK-99 and BK-100 are now closed — BK-16 can be unblocked.

---


_Synced from Jira by sync-jira-issues_
