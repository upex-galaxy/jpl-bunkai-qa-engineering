# Comments for BK-60

[View in Jira](https://jira.upexgalaxy.com/browse/BK-60)

---

### Nahuel Gomez - 5/6/2026, 19:36:23

## Test Evidence — BK-60

### Repro Steps

```bash
# 1. Get workspace ID (workspace aed86386 has owner qa-headless@bunkai.io)
curl -H 'Authorization: Bearer bk*pat*ZBOc...' \
  https://staging-upexbunkai.vercel.app/api/v1/workspaces

# 2. Owner invites themselves (email already a member)
curl -X POST https://staging-upexbunkai.vercel.app/api/v1/workspaces/aed86386-2ed8-424e-934b-ca7a0ef6af37/invites \
  -H 'content-type: application/json' \
  -d '{"email":"qa-headless@bunkai.io","role":"member"}'
```

### Actual (BUG): 201 Created

```json
{
  "invite": {
    "id": "bbb9a656-8f86-4ff4-bd97-2acacfc9d1c1",
    "workspace_id": "aed86386-2ed8-424e-934b-ca7a0ef6af37",
    "email": "qa-headless@bunkai.io",
    "role": "member",
    "status": "pending"
  },
  "token": "bk*inv*2rTgTxbLC5R21dcL6WpGX",
  "accept*url": "/invites/accept?token=bk*inv_2rTgTxbLC5R21dcL6WpGX"
}
```

### Expected: 409 Conflict

```json
{
  "error": "EMAIL*ALREADY*MEMBER",
  "message": "This email already belongs to an active workspace member"
}
```

### DB Evidence

`qa-headless@bunkai.io` IS already a member of workspace `aed86386`:

```sql
SELECT email, role FROM workspace*members WHERE workspace*id = 'aed86386-2ed8-424e-934b-ca7a0ef6af37';
-- email: qa-headless@bunkai.io, role: member (was owner, demoted by BUG-CRIT-3 BK-62)
```

### Spec Reference

FR-003 Business Rule: "email MUST be unique among active workspace members."

---

### Automation for Jira - 10/6/2026, 18:24:37

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 10/6/2026, 18:24:46

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 10/6/2026, 18:28:05

## 🔧 Bug Fix Documentation

### Root Cause Analysis

***Category******:*** Code Error
***Location******:*** `app/api/v1/workspaces/[id]/invites/route.ts` — POST handler

The handler inserted the invite with ***zero uniqueness checks*** — no query against `workspace*members` before insert. Note: `workspace*members` has no email column; mapping email→user requires `auth.users`, which PostgREST does not expose — likely why the check was skipped originally.

### Fix Applied

- New SECURITY DEFINER fn `bunkai*user*id*by*email` (migration 0022, callable by service role only) resolves the email without exposing `auth.users`.
- POST now returns ***409 ***`email*already*member` ("This email already belongs to an active workspace member.") when the email maps to an ACTIVE member — per FR-003.
- The admin/owner gate runs BEFORE the uniqueness probes, so non-admins cannot use the endpoint to discover membership facts.

***PR******:**** https://github.com/upex-galaxy/upex-bunkai-tms/pull/34 (merged to `staging`, commit `8c67211`, deployed) · ****Fix Type******:*** Bugfix · Gates: `bun test` 192/192 (8 new unit tests), `tsc` clean, `eslint` clean.

### Verification Performed (staging, post-deploy)

- [x] QA repro: invite issued for an active member's email → ***HTTP 409 ***`{"code":"conflict","reason":"email*already*member"}` (was 201 + token leak)

### How to Verify

1. As workspace admin/owner, POST `/api/v1/workspaces/{id}/invites` with the email of an existing ACTIVE member
2. ***Expected******:*** 409 with `details.reason = email*already*member`; no invite row, no token

---

**Fix ready for QA verification.**

---

### Nahuel Gomez - 10/6/2026, 20:22:00

## QA Retest: PASSED ✓

***Retest date***: 2026-06-10
***Environment***: staging (https://staging-upexbunkai.vercel.app)
***Tester***: qa-headless@bunkai.io

### Verification
POST /api/v1/workspaces/{id}/invites with email of existing active member now correctly returns HTTP 409 with error code "conflict" and message "This email already belongs to an active workspace member."

### Evidence
- POST /workspaces/dfdd3fb7.../invites {email:"qa-headless@bunkai.io", role:"member"} → 409 ✓
- Error code: conflict, reason: email*already*member ✓

### Verdict
Bug fixed. Transitioning to Closed.

---


_Synced from Jira by sync-jira-issues_
