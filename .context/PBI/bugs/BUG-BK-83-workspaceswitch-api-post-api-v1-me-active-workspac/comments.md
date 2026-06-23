# Comments for BK-83

[View in Jira](https://jira.upexgalaxy.com/browse/BK-83)

---

### Automation for Jira - 10/6/2026, 12:42:56

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 10/6/2026, 12:43:28

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 10/6/2026, 12:46:22

## 🔧 Bug Fix Documentation

### Root Cause Analysis

***Category******:*** Code Error
***Location******:*** `app/api/v1/me/active-workspace/route.ts` — POST handler

The handler validated membership with a bare `select('id')` on `workspaces`, set the `bk*active*ws` cookie, and responded `{ ok, active*workspace*id }`. It never queried the workspace's `slug`/`name` nor the caller's `role`, so the BK-6 AC1 response contract `{ id, slug, name, role }` was unmet and UI consumers needed a follow-up `GET /me`.

### Fix Applied

***Branch******:*** `fix/BK-83-active-workspace-response`
***PR******:*** https://github.com/upex-galaxy/upex-bunkai-tms/pull/32 (merged to `staging`, commit `ccf5e52`, deployed)
***Fix Type******:*** Bugfix

| File | Change |
| --- | --- |
| `app/api/v1/me/active-workspace/route.ts` | Membership check now selects `id, slug, name`; caller `role` fetched from `workspace*members` via the RLS-scoped client (self-select policy); response is a flat superset `{ ok, active*workspace_id, id, slug, name, role }` |
| `app/api/v1/me/active-workspace/route.openapi.ts` | `ActiveWorkspaceResponse` schema +4 fields |
| `public/openapi.json` | Regenerated |

Non-breaking: both existing consumers (`WorkspaceSwitcher.tsx`, `AppSidebar.tsx`) only read `res.ok`; the original `ok`/`active*workspace*id` keys are preserved.

### Verification Performed

- [x] `bun test` 180/180 · `tsc --noEmit` clean · `eslint` clean
- [x] Staging smoke (2026-06-10, post-deploy): `POST /api/v1/me/active-workspace` → `HTTP 200` with body

  `{"ok":true,"active*workspace*id":"…","id":"…","slug":"bunkai-smoke-qa","name":"Bunkai Smoke QA","role":"owner"}`

- [x] Membership-denial path unchanged (403 forbidden)
- [x] OpenAPI on staging exposes the new schema (`required: [ok, active*workspace*id, id, slug, name, role]`)

### How to Verify

1. Authenticate on staging as a member of 2+ workspaces
2. `POST /api/v1/me/active-workspace` with `{"workspace_id":"<other-workspace>"}`
3. ***Expected******:*** HTTP 200 body contains `id`, `slug`, `name`, `role` of the NEW workspace (plus legacy `ok` + `active*workspace*id`)
4. This unblocks story ***BK-6*** (AC1 — Successful workspace switch)

---

**Fix ready for QA verification.**

---

### Luis Eduardo Flores Villarroel - 11/6/2026, 23:19:21

## Acceptance Test Results (ATR)

***BK-83 TEST RESULTS***

| Field | Value |
| --- | --- |
| Tested | 2026-06-12 |
| Environment | Staging (https://staging-upexbunkai.vercel.app) |
| Result | ***PASSED*** (4/4 TCs — 100%) |

---

## Summary

Retested fix for POST /api/v1/me/active-workspace. The endpoint now returns the required fields `{id, slug, name, role}` as specified in the ATP. All 4 test cases passed. Fix is confirmed on staging.

Legacy fields `ok` and `active*workspace*id` remain in the response (additive, non-breaking). Cleanup tracked as separate tech debt ticket.

---

## Test Cases

| TC | Priority | Description | Result |
| --- | --- | --- | --- |
| TC1 | P0 | POST /api/v1/me/active-workspace returns {id, slug, name, role} | PASSED |
| TC2 | P0 | bk*active*ws cookie set on successful workspace switch | PASSED |
| TC3 | P1 | Non-member workspace returns 403 Forbidden | PASSED |
| TC4 | P1 | Switched workspace reflected in state consistency check | PASSED |

---

## Test Data

| Entity | Name | ID |
| --- | --- | --- |
| Workspace (switched to) | extra-test | (UUID from response) |
| Workspace (ownership test) | bk5-test-ws | c828d131 |
| Non-member workspace | first-smoke-test | (no membership row) |

---

## TC Detail

***TC1*** — Required fields confirmed: `id`, `slug` ("extra-test"), `name` ("Extra Test"), `role` ("member"). DB cross-validation: role matches `workspace_members` table.

***TC2*** — Cookie: `bk*active*ws=a808499e...; Path=/; Max-Age=7776000; Secure; HttpOnly; SameSite=lax`. All security attributes correct.

***TC3*** — Non-member workspace (first-smoke-test): 403 enforced. Response: `{"error":{"code":"forbidden","message":"You are not a member of that workspace."}}`.

***TC4*** — Switched to bk5-test-ws. POST `.id` matches requested workspace*id. Cookie `bk*active_ws` matches. GET /me via PAT returns original workspace by design (bearer ignores cookie per `me/route.ts`).

***DB Validation*** — Membership + roles confirmed for 3 workspaces. Non-member confirmed. No `active*workspace*id` column — cookie-only state by design.

---

## Bugs Found

None blocking.

> ***WARNING:*** Legacy fields `ok` and `active*workspace*id` still present in response alongside `{id, slug, name, role}`. Additive, non-breaking. Tech debt ticket filed separately.

---

## Observations

- Legacy response fields (`ok: true`, `active*workspace*id`) not removed from `route.ts`. Tech debt tracked separately.
- `bun run api:login:staging` exits code 1 (key mismatch: expects `access_token`, gets `pat.token`). Non-blocking — existing PAT valid.
- Local `route.ts` unpatched (old shape). Staging deployment has the fix.

---

## Recommendations

- Automate TC1 + TC2 as regression ATCs (API contract + cookie). Stage 5 candidate.
- Address legacy field cleanup before next API consumer onboarding.

---

### Luis Eduardo Flores Villarroel - 11/6/2026, 23:19:52

## QA Bug Verification - BK-83

| Field | Value |
| --- | --- |
| Environment | Staging (https://staging-upexbunkai.vercel.app) |
| Date | 2026-06-12 |
| Result | ***VERIFIED*** — Bug fix confirmed |
| TCs | 4/4 PASSED (100%) |

---

## Verification

***Original bug scenario******:*** No longer reproduces. POST /api/v1/me/active-workspace now returns `{id, slug, name, role}` as required.

***Expected behavior confirmed******:***

- TC1: Response shape `{id, slug, name, role}` — present and correct. Role matches `workspace_members` DB.
- TC2: `bk*active*ws` cookie set on switch — `Secure; HttpOnly; SameSite=lax` attributes all correct.
- TC3: 403 enforced for non-member workspace — `You are not a member of that workspace.`
- TC4: POST response and cookie internally consistent after switch.

***Regression check******:*** No issues found. DB validation confirmed membership and role accuracy.

---

## Non-blocking Observation

> ***WARNING:*** Legacy fields `ok: true` and `active*workspace*id` still present in response alongside the fix fields. Additive — no existing consumer is broken. A separate tech debt ticket has been filed for cleanup (see related issues).

---

## Artifacts

| Artifact | Reference |
| --- | --- |
| ATP | BK-83 field `customfield_10120` (written 2026-06-12) |
| ATR | BK-83 comment #11564 |
| Evidence | `.context/PBI/bug/BK-83/evidence/` (5 files) |

---


_Synced from Jira by sync-jira-issues_
