# Comments for BK-57

[View in Jira](https://jira.upexgalaxy.com/browse/BK-57)

---

### Automation for Jira - 10/6/2026, 20:50:38

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 10/6/2026, 20:50:54

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 10/6/2026, 20:53:50

## Root Cause Analysis

`PATCH /api/v1/modules/{id}` accepted rename/description fields AND `parent*module*id` in one body, then executed them as TWO independent RPC round-trips (`bunkai*update*module`, then `bunkai*move*module`). Each RPC is atomic internally, but the two calls are separate PostgREST requests = separate transactions. If the move failed (45003 parent*invalid, 45001 move*cycle, 45002 depth_exceeded, or a path collision) after the update committed, the client received an error envelope while the rename/description change had already persisted — a half-applied PATCH. Exposure was API-only: the UI always sends rename and move as separate requests.

## Fix Applied

Ratified option: ***reject the combined request*** (the alternative — a new atomic combined RPC — was declined as non-minimal).

- New pure helper `modulePatchShapeError({hasName, hasDescription, hasParent})` in `lib/modules/validation.ts` (also replaces the inline `no_fields` guard; presence semantics unchanged — `description: null` still means "clear").
- Combined rename/description+move → 422 `validation*failed`, `details.reason: 'combined*update*and*move'`, message: "Rename/description edits and a move cannot be combined in one request — send them as separate PATCH calls."
- OpenAPI updated: `parent*module*id` documented as mutually exclusive with `name`/`description` (field, operation, and 422 descriptions). Note: this narrows the previously advertised contract — external bearer-token consumers combining the fields now get 422 by design.

PR #37 (merged to staging, deploy verified).

## Verification Performed

- Unit: full shape matrix in `lib/modules/validation.test.ts` (no_fields / each field alone / rename+description legal / every combined-with-move shape rejected). Suite 237/237 green.
- Staging smoke (zero data writes): PATCH with `{name, parent*module*id}` → 422 `combined*update*and_move`; single-purpose rename PATCHes still 200.

## How to Verify

PATCH a module with `{"name":"X","parent*module*id":"<other-module-id>"}` → 422 `combined*update*and_move`, and confirm neither the rename nor the move persisted. Separate rename-only and move-only PATCHes keep working.

**Fix ready for QA verification.**

---


_Synced from Jira by sync-jira-issues_
