# Comments for BK-62

[View in Jira](https://jira.upexgalaxy.com/browse/BK-62)

---

### Nahuel Gomez - 5/6/2026, 19:36:41

## Test Evidence — BK-62

### Repro Steps

```bash
# 1. Owner creates invite for member role
curl -X POST https://staging-upexbunkai.vercel.app/api/v1/workspaces/aed86386-2ed8-424e-934b-ca7a0ef6af37/invites \
  -H 'content-type: application/json' \
  -d '{"email":"qa-headless@bunkai.io","role":"member"}'
# → 201, token: bk*inv*2rTgTxbLC5R21dcL6WpGX

# 2. Owner accepts the member-role invite
curl -X POST https://staging-upexbunkai.vercel.app/api/v1/invites/accept \
  -H 'content-type: application/json' \
  -d '{"token":"bk*inv*2rTgTxbLC5R21dcL6WpGX"}'
# → 201 Accepted

# 3. Check role in DB — OWNER DEMOTED TO MEMBER
```

### Actual (BUG): Role overwritten to member

```sql
-- BEFORE accept:
SELECT user*id, role FROM workspace*members
WHERE workspace_id = 'aed86386-2ed8-424e-934b-ca7a0ef6af37'
  AND email = 'qa-headless@bunkai.io';
-- role: owner ✅

-- AFTER accept:
SELECT user*id, role FROM workspace*members
WHERE workspace_id = 'aed86386-2ed8-424e-934b-ca7a0ef6af37'
  AND email = 'qa-headless@bunkai.io';
-- role: member ❌  (DEMOTED!)
```

### DB Evidence (live — staging, workspace aed86386)

```sql
SELECT wm.role, wi.role AS invited_role
FROM workspace_members wm
JOIN workspace*invites wi ON wi.workspace*id = wm.workspace_id
WHERE wm.workspace_id = 'aed86386-2ed8-424e-934b-ca7a0ef6af37'
  AND wm.email = 'qa-headless@bunkai.io';

-- Result: wm.role = 'member', wi.role = 'member'
-- Owner was demoted from 'owner' to 'member' by the accept upsert
```

### Root Cause — `app/api/v1/invites/accept/route.ts:77-87`

```typescript
// PROBLEM: upsert sets role unconditionally
const { data: member, error } = await supabase
  .from("workspace_members")
  .upsert({
    workspace_id,
    user_id: authUser.id,
    role: invite.role,  // ← ALWAYS overwrites, even if user is owner/admin
    status: "active",
  }, { onConflict: "workspace*id,user*id" })
```

### Impact

- ***Data integrity breach***: workspace `aed86386` owner permanently demoted to member
- No way to undo via API (no role-change endpoint)
- Requires manual DB fix on staging

### Fix

Check existing membership before upsert:

```typescript
const {data: existing} = await supabase.from("workspace_members")
  .select("role").eq("workspace*id", workspace*id).eq("user_id", authUser.id).single();

if (existing && roleHierarchy[existing.role] >= roleHierarchy[invite.role]) {
  return error(409, "ALREADY_MEMBER", "You are already a member with equal or higher role");
}
// else upsert with NEW role
```

---

### Automation for Jira - 10/6/2026, 18:24:37

🔎 Pull Request created. Task is pending to ANALYZE and REVIEW by the team. Waiting for PR Approval.

---

### Automation for Jira - 10/6/2026, 18:24:46

✅ Pull Request is successfully MERGED. Task is Done.

---

### Ely - 10/6/2026, 18:28:07

## 🔧 Bug Fix Documentation

### Root Cause Analysis

***Category******:*** Code Error
***Location******:*** `app/api/v1/invites/accept/route.ts` — membership upsert

The accept handler ran an ***unconditional*** `workspace_members.upsert` with `role = invite.role` — any existing row (including an owner's) was overwritten with the invite's role. Exactly the reported demotion: owner accepts a member-role invite → owner becomes member.

### Fix Applied

- New pure decision helper `inviteAcceptAction` (`lib/workspaces/invites.ts`, rank: viewer < member < admin < owner):
- 8 unit tests including the exact owner-demotion repro.
- Defense in depth: BK-60's fix prevents issuing invites to active members in the first place, sealing the demotion vector at both ends.

***PR******:**** https://github.com/upex-galaxy/upex-bunkai-tms/pull/34 (merged to `staging`, commit `8c67211`, deployed) · ****Fix Type******:*** Bugfix · Gates: `bun test` 192/192 (8 new unit tests), `tsc` clean, `eslint` clean.

### Data Repair (staging)

Workspace `aed86386-2ed8-424e-934b-ca7a0ef6af37` ("QA Test Workspace") — the damaged owner row was restored ***member → owner*** via a 1-row UPDATE keyed on `workspaces.owner*user*id` (verified with RETURNING). The reported damage is undone.

### Verification Performed

- [x] Unit repro: `inviteAcceptAction({role:'owner',status:'active'}, 'member')` → `reject*already*member` (test green)
- [x] Equal role → reject · member→admin promotion → allowed · `status='invited'` activation → allowed (all tested)
- [x] Staging owner row of aed86386 verified `role='owner'` after repair

### How to Verify

1. With a legacy/pre-existing invite addressed to an active member (new ones are now blocked by BK-60), accept it as that member
2. ***Expected******:*** 409 `already*member*equal*or*higher_role`; the member's role is unchanged
3. Verify the aed86386 workspace owner can manage the workspace again (role restored)

---

**Fix ready for QA verification.**

---

### Nahuel Gomez - 10/6/2026, 20:22:20

## QA Retest: MITIGATED ✓ (exploit path blocked by BK-60 fix)

***Retest date***: 2026-06-10
***Environment***: staging

### Analysis
Original exploit (owner accepts lower-role invite → demoted) can no longer be triggered. BK-60 now rejects invite creation for emails that already belong to active workspace members (409 EMAIL*ALREADY*MEMBER).

Attempted POST /workspaces/aed86386.../invites {email:"qa-headless@bunkai.io", role:"member"} → 409 ✓

### Code review note
The accept endpoint still uses unconditional `role: invite.role` in `workspace_members.upsert`. This is not immediately exploitable (BK-60 gates prevent reaching this path with a lower role), but represents missing defense-in-depth.

### Recommendation
Close this bug (exploit path blocked) but consider a follow-up tech-debt ticket to add a server-side guard in the accept endpoint: check existing membership role before upsert and preserve the higher role.

### Verdict
Bug mitigated via BK-60. Closing.

---


_Synced from Jira by sync-jira-issues_
