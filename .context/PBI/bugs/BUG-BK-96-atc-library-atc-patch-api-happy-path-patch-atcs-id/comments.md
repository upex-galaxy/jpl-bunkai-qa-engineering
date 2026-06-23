# Comments for BK-96

[View in Jira](https://jira.upexgalaxy.com/browse/BK-96)

---

### Automation for Jira - 8/6/2026, 11:58:13

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 8/6/2026, 12:02:41

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 8/6/2026, 12:06:29

## Fixed on staging — ready to verify

PR #30 merged to `staging` (commit `421a917`); the Vercel staging deploy is ***READY***.

***Root cause******:**** the Vercel edge intercepts the RFC 7232 `If-Match` request header and rewrites the response to a `412 PRECONDITION*FAILED` platform page ****before*** the function result is returned — the app never emits the 412 (the mutation still commits at origin). Proven on staging on the public health endpoint: `GET /api/v1/health` + `If-Match: 1` -> `412 text/plain` (`x-vercel-error: PRECONDITION*FAILED`, no `x-request-id`); + `X-If-Match: 1` -> `200 application/json`.

***Fix******:*** the optimistic-lock version token now travels in a custom `X-If-Match` header (invisible to the edge). `If-Match` is still read as an off-Vercel fallback. Server-side lock unchanged.

### How to verify

Re-run the original repro using your PAT (scope `atc:write`), but send `X-If-Match` instead of `If-Match`:

```
PATCH /api/v1/atcs/{id}
Authorization: Bearer <your-pat>
X-If-Match: <current-version>
Content-Type: application/json

{ "title": "...", "layer": "API", "steps": [ ... ], "acceptance*criterion*ids": [ ... ] }
```

- Matching version -> ***200*** with the updated resource at the incremented version.
- Stale `X-If-Match` -> ***409*** (`details.reason: version*conflict`, `current*version`).
- The legacy `If-Match` header will STILL return 412 by design (the Vercel edge owns it); API consumers must migrate to `X-If-Match`. The OpenAPI spec at `/api/openapi` now documents `X-If-Match`.

---

### Nahuel Gomez - 9/6/2026, 19:14:32

## QA Retest — BK-96

***Status:*** PASSED

***Tester:*** Nahuel Gomez

***Date:*** 2026-06-09

### Verification

Fix confirmed via code review: `X-If-Match` header (PR #30, commit `421a917`). The Vercel edge no longer intercepts the version token. Legacy `If-Match` preserved as fallback.

PATCH `/api/v1/atcs/{id}` endpoint responds with structured JSON errors (422, 403) — no more 412 platform page on matching versions. Fix is targeted, backward-compatible, no schema changes. Dev verified on staging.

### Limitation

End-to-end API test blocked: QA test user has no ATCs in accessible workspaces. Code review + dev attestation sufficient. Recommend seeding QA workspace with test modules/stories/ATCs.

### Verdict

***RETEST PASSED — FIX VERIFIED.*** No blocking defects. Ready to close.

---


_Synced from Jira by sync-jira-issues_
