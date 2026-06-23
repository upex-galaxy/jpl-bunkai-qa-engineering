# Comments for BK-51

[View in Jira](https://jira.upexgalaxy.com/browse/BK-51)

---

### Automation for Jira - 10/6/2026, 20:36:43

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 10/6/2026, 20:37:09

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 10/6/2026, 20:41:22

## Root Cause Analysis

POST `/api/v1/workspaces/{id}/projects` had no reserved-slug guard anywhere in its pipeline — the only such guard lived in the workspace create route and was never replicated or shared. The AC-11 agreed 18-word reserved list (ratified in the BK-8 Dev Q8 comment) was never implemented, so names like `api` / `new` / `settings` passed every existing check and returned 201.

## Fix Applied

- New pure helper `lib/projects/validation.ts`: `RESERVED*PROJECT*SLUGS` (the exact AC-11 18-word list — deliberately separate from the workspace route's list, different collision surface) + `isReservedProjectSlug()`.
- The route now rejects on the ***final derived slug*** (so `API`, ` New `, `Settings!` are caught) with 422 `validation*failed` and `details.reason: 'slug*reserved'` — the route's established hybrid error model. Note: the Dev Q8 comment had proposed a top-level `400 SLUG_RESERVED`; the shipped shape follows this bug's Expected Result and the route's contract.
- Create-project form maps `slug_reserved` to the design-ratified copy: "This name is reserved. Try a different one."
- OpenAPI descriptions updated.

PR #36 (merged to staging, deploy verified).

## Verification Performed

- Unit: table-driven suite over all 18 words + slugify-derivation cases + near-miss negatives (`api-tests`, `newest`, …) + list-parity guard (`size === 18`). Full suite 227/227 green.
- Staging smoke (zero residue — 422s create nothing): `{"name":"api"}` → 422 `slug*reserved`; `{"name":"Settings!"}` → 422 `slug*reserved`.

## How to Verify

POST `/api/v1/workspaces/{your-ws-id}/projects` with `{"name":"api"}` (or any AC-11 word, any casing/punctuation that derives it) → 422 with `details.reason = "slug_reserved"`; the create-project form shows the reserved-name message. Non-reserved names still create normally.

**Fix ready for QA verification.**

---


_Synced from Jira by sync-jira-issues_
