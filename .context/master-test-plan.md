# Master Test Plan — Bunkai TMS

> Generated: 2026-06-23
> Command: `/master-test-plan`
> Sources: business-data-map.md · business-feature-map.md · business-api-map.md
> Re-generate after: new epic ships, major migration, new integration added

---

```
+============================================================+
|  Bunkai TMS — Master Test Plan                             |
|  What to test in this system, and why it matters           |
+============================================================+
```

---

## Section 1 — Executive Risk Map

Bunkai TMS is a multi-tenant test management platform where the entire data model — every project, module, ATC, and result — is scoped to a workspace, isolated by Supabase Row-Level Security. That makes RLS the single highest-stakes layer in the system: a gap in any policy silently exposes one tenant's data to another, which is not just a bug but a breach. On top of that isolation layer, the core product value lives in the ATC Save flow — the only mechanism that persists test cases — and it relies on an atomic PostgreSQL RPC whose internals have not been independently verified. The invitation and PAT flows carry their own security weight: both involve one-time secrets shown only once in the API response, meaning a bug in token validation or revocation checking has no recovery path. Finally, the entire run-execution and ATC status-transition model is unimplemented — the `runs` entity does not exist yet — which means the most visible, most user-facing half of a TMS (actually running tests and recording results) is a future state that will need end-to-end testing from scratch when it ships.

| Priority | Flow | Why it matters | Depends on / Affects |
|----------|------|----------------|----------------------|
| CRITICAL | Magic Link Auth + Session (FEAT-001, FEAT-006, FR-006) | Entry gate — no user can reach any other flow without a valid session cookie | Supabase Auth, Resend OTP delivery, middleware session refresh |
| CRITICAL | Workspace Bootstrap — `bunkai_bootstrap_workspace` RPC (FEAT-007, FR-001) | Atomic SECURITY DEFINER; failure leaves workspace row with no owner member — orphaned and inaccessible forever | All downstream entities scoped to workspace |
| CRITICAL | ATC Save — `bunkai_save_atc` RPC (FEAT-022, FR-004) | Core product value; atomic upsert of 4 sub-record types; RPC internals unverified; viewer write-block relies entirely on unconfirmed RLS | atc_steps, atc_assertions, atc_acceptance_criteria, ISR cache |
| CRITICAL | Supabase RLS — cross-workspace isolation (BR-002, BR-003, BR-050) | A misconfiguration silently returns another tenant's ATCs, members, or invites; no app-layer fallback | Every table; every API endpoint |
| HIGH | Invite Issue + Accept lifecycle (FEAT-011, FEAT-015, FR-002, FR-003) | One-time token shown once; no email delivery in MVP; expired, revoked, and already-accepted states each have distinct rejection logic that must all work | workspace_invites, workspace_invite_secrets, workspace_members |
| HIGH | PAT Mint + Bearer Auth verification (FEAT-002, FEAT-016, FEAT-018, FR-005) | CI pipelines depend on valid Bearer tokens; soft revoke must block immediately; role-gate gap allows viewer to issue `atc:write` scope | access_tokens, access_token_secrets, all Bearer-authenticated endpoints |
| HIGH | Headless CI Bootstrap — signup + PAT in one call (FEAT-002, FEAT-003) | Primary non-interactive entry point for automation; 409 idempotency must be reliable; uniform 401 must not leak email existence | auth.users, mintPat(), all CI-gated flows |
| HIGH | Viewer write-block via RLS on `atcs` (BR-003, BR-009) | `saveAtcAction` does not check role at app layer; entire write guard is RLS-only on unconfirmed `atcs_member_plus` policy | atcs, atc_steps, atc_assertions |
| MEDIUM | ATC Fulltext Search + GIN index (FEAT-023, FR-007, BR-050) | Cross-workspace search leak would be a tenant isolation breach; `tsv` trigger must fire on every title/tag update | atcs.tsv, bunkai_atcs_refresh_tsv trigger |
| MEDIUM | Active Workspace Switch — `bk_active_ws` cookie (FEAT-010, FR-008) | Cookie staleness or membership bypass silently corrupts the user's data scope for all subsequent requests | bk_active_ws cookie, /me introspection, workspace_members RLS |

---

## Section 2 — What to Test First and Why

### 2.1 Magic Link Auth + Session (CRITICAL)

**Why it matters.** Every QA engineer, admin, and stakeholder who opens Bunkai lands on the login page first. If the magic link OTP doesn't arrive, nobody enters the system. If the session cookie is misconfigured or expires unexpectedly, authenticated users get silently bounced to `/login` mid-session, losing work in progress. Because Bunkai has no password reset and no fallback login today, an auth failure is a total lockout — not a degraded experience.

**What commonly breaks.** Resend SMTP delivery through Supabase Auth can fail silently — the API call succeeds (200) but the email never arrives. The `?next=` redirect parameter needs to survive the multi-hop flow (magic-link dispatch → email click → `/auth/callback` → final redirect); a URL-encoding bug anywhere in that chain drops the user at an unexpected route. The open-redirect guard on `next` must reject values starting with `//` while accepting `/` paths — edge cases here are classic targets. Cookie refresh in `middleware.ts` on every request means a stale-cookie race condition (token refresh fails while user is mid-form) can produce a 401 at the worst possible moment.

**Dependencies.** Supabase Auth OTP engine → Resend SMTP relay → `/auth/callback` PKCE exchange → `sb-<ref>-auth-token` httpOnly cookie → `middleware.ts` `getUser()` on every protected request.

**What an experienced QA would check.**

- Send magic link to a valid email, click within the OTP window, confirm session cookie is set and user lands at the correct `next` path.
- Attempt to reuse an already-consumed OTP link and verify Supabase Auth rejects it (BR-040 — Bunkai's own `consumed_at` is never stamped, so this relies entirely on Supabase internals; you're probing that internal guarantee).
- Verify the `next` redirect guard: `next=//evil.com` must be ignored; `next=/projects` must survive the full callback chain.
- Let the session expire (or delete the cookie) mid-navigation and confirm middleware redirects to `/login?next=<original path>` cleanly.
- Simulate Resend delivery failure in staging and confirm the magic-link endpoint returns a user-facing error rather than a silent 200.

---

### 2.2 Workspace Bootstrap — `bunkai_bootstrap_workspace` RPC (CRITICAL)

**Why it matters.** Every other entity in Bunkai hangs off a workspace. The bootstrap RPC is SECURITY DEFINER, which means it runs with elevated privileges to break the chicken-and-egg problem: a user can't insert a workspace row (RLS requires active membership) until the workspace exists. If this RPC partially fails — creates the workspace row but not the `workspace_members` owner row — the workspace is orphaned: inaccessible, unrecoverable without a DB-level intervention, and the user sees an error with no retry path.

**What commonly breaks.** Slug uniqueness collisions return a 409, but the client gets no slug suggestions, leaving the user stuck. Reserved-slug validation uses an 18-word list checked at the application layer; a slug that embeds a reserved word (e.g., `my-api-project`) might pass while a direct match (`api`) is rejected — the boundary here needs testing. The slug regex allows 3–40 chars with a specific charset — boundary violations at length 2, 41, and with invalid characters like uppercase or leading/trailing hyphens are classic miss zones.

**Dependencies.** `bunkai_bootstrap_workspace` PostgreSQL RPC (SECURITY DEFINER) → atomic INSERT into `workspaces` + `workspace_members` → 201 response → redirect to `/projects`.

**What an experienced QA would check.**

- Valid slug + valid name → workspace created, caller has `role=owner, status=active` in `workspace_members`.
- Slug at boundary lengths: 2 chars (rejected), 3 chars (accepted), 40 chars (accepted), 41 chars (rejected).
- Slug with a reserved word (`admin`, `api`, `login`) → rejected; slug with a reserved word embedded (`my-api-project`) → accepted.
- Duplicate slug → 409, no partial workspace row left behind; query DB to confirm.
- Unauthenticated call → RPC raises `42501`; no workspace row created.
- Happy-path atomicity: confirm both `workspaces` and `workspace_members` rows exist after success — DB-level check, not just UI response.

---

### 2.3 ATC Save — `bunkai_save_atc` RPC (CRITICAL)

**Why it matters.** Saving an ATC is the primary daily action for every QA engineer in Bunkai. A single save persists up to four entity types atomically (the ATC itself, its steps, its assertions, and its AC links) via a SECURITY INVOKER RPC. If the atomicity breaks — steps saved but assertions rolled back, or AC links dropped — the ATC is in a corrupted state with no visible error. The "anchoring moat" (BR-021) is the defining product constraint: an ATC with zero AC links is not a valid test case in Bunkai's model.

**What commonly breaks.** The viewer write-block is NOT in `saveAtcAction` — a viewer who reaches the save action relies entirely on Supabase RLS to get rejected; if that policy has a gap, a viewer silently overwrites another engineer's ATC. Markdown parsing via `parseStepsMarkdown()` and YAML parsing via `parseAssertionsYaml()` can produce empty arrays for valid-looking content (wrong heading level, YAML indent error); the save succeeds but the ATC has no steps or no assertions. ISR cache revalidation via `revalidatePath()` runs after save — if it fails, the project tree shows stale data.

**Dependencies.** `saveAtcAction` (Server Action) → app validation → `parseStepsMarkdown()` → `parseAssertionsYaml()` → `bunkai_save_atc` RPC (SECURITY INVOKER, inherits caller RLS) → atomic upsert of `atcs` + `atc_steps` + `atc_assertions` + `atc_acceptance_criteria` → `revalidatePath()` for editor + project tree.

**What an experienced QA would check.**

- Full happy-path save with title, layer, tags, one story, two ACs, three steps, and one assertion — verify all four sub-record types exist in DB.
- Save with zero AC links (`acIds = []`) → app returns `"Bind at least one acceptance criterion."` and nothing is written.
- Save with blank title (`"  "` after trim) → app returns `"Title is required."`.
- Viewer role calling save → RLS blocks at DB; confirm the server action surfaces a meaningful error rather than swallowing the exception.
- Partial content: steps with valid Markdown but assertions left empty → `atc_assertions` produces zero rows, not a stale prior state.
- Re-save (second save on same ATC) → steps are replaced cleanly, not duplicated.

---

### 2.4 Supabase RLS — Cross-Workspace Isolation (CRITICAL)

**Why it matters.** Bunkai is a multi-tenant SaaS. Every RLS policy on every one of the 12 data tables subqueries `workspace_members` for the caller's active membership before allowing any read or write. A single misconfigured policy leaks one workspace's data to another user's session. This is the highest-blast-radius failure in the system because it affects all users simultaneously, is invisible in the UI, and the only way to detect it is to run cross-tenant probes.

**What commonly breaks.** RLS policies that check `workspace_id` via a JOIN can silently return empty results rather than 403 when the caller is not a member — the UI shows "no data" and nobody investigates. Fulltext search via the GIN index (`atcs.tsv`) must also be scoped by RLS (BR-050). Suspended members (`status = 'suspended'`) must be treated the same as non-members by every policy — if any policy checks only role and not status, a suspended admin can still read data.

**Dependencies.** All 12 DB tables → RLS policies on each → `workspace_members` subquery for membership + role + status → behavior visible in every API endpoint.

**What an experienced QA would check.**

- Two users, two workspaces: user A calls `GET /api/v1/workspaces` and must not see workspace B; user B's ATCs must not appear in user A's project tree.
- Suspended member attempts any read → response must be empty or 403; not partial data.
- ATC fulltext search from workspace A with a keyword known to exist only in workspace B's ATCs → zero results.
- Bearer PAT scoped to workspace A must not read workspace B's data, even if the underlying user is a member of both.
- Viewer in workspace A attempts ATC save → RLS blocks at DB; no `atcs` row modified.

---

### 2.5 Invite Issue + Accept Lifecycle (HIGH)

**Why it matters.** Inviting a teammate is the only way to grow a workspace team. The invite token is shown exactly once in the API response — if the admin closes the browser tab, there's no recovery path until the token is rotated. The accept flow enforces email match (BR-015), expiry (BR-013), one-time use (BR-014), and revocation (BR-016) — four independent validation checks that must each work because any failure constitutes an access-control breach.

**What commonly breaks.** The owner role exclusion (BR-010) must be enforced at both the Zod schema level and the DB CHECK. Token rotation (FEAT-013) clears `accepted_at` and `revoked_at`, effectively reactivating an invite row — you'll want to verify that a rotated invite for an already-accepted member doesn't create a duplicate `workspace_members` row.

**Dependencies.** `workspace_invite_secrets` (service_role-only table for hash storage) → email match against `auth.users.email` → UPSERT `workspace_members` → stamp `accepted_at` on invite row.

**What an experienced QA would check.**

- Admin issues invite → captures token from response → invitee authenticates with matching email → accepts → `workspace_members` row created with correct role and `status=active`.
- Invitee with wrong email → 403; no membership row created.
- Expired token → 409; membership not created.
- Already-accepted token used again → 409; second membership row not created.
- Revoked invite → 409; membership not created.
- Token rotation: admin rotates an expired invite → new token works, old token does not.

---

### 2.6 PAT Mint + Bearer Auth Verification (HIGH)

**Why it matters.** CI pipelines and QA agents depend on Personal Access Tokens for all headless access. The token is shown once at creation (BR-032) and is never recoverable. Two active security gaps make this flow higher risk than it looks: a `viewer` can issue a PAT with `atc:write` scope because the mint endpoint is session-gated, not role-gated; and the bearer middleware hashes `prefix + remainder` (the full secret) — an earlier bug hashed only `remainder` causing perpetual 401s, worth independently validating.

**What commonly breaks.** Soft revoke must take effect on the very next Bearer request — there's no caching layer on the revocation check. Expired tokens (`expires_at < now()`) must return 401 immediately. The scope check via `requireScopeOrCookie()` must reject a token missing the required scope.

**Dependencies.** `mintPat()` → `access_tokens` (prefix + scopes) + `access_token_secrets` (SHA-256 hash, service_role-only) → `requireBearerToken()` middleware → O(1) prefix lookup → constant-time SHA-256 comparison → revocation + expiry checks → scope enforcement.

**What an experienced QA would check.**

- Mint a PAT with `atc:read` scope, use it on a read endpoint → 200; use it on a write endpoint → 403.
- Soft-revoke the token → immediately call a read endpoint with the revoked token → 401.
- Mint with `expires_in_days = 1`, call endpoint within expiry → 200; call after expiry → 401.
- Issue a PAT as a `viewer`-role user with `atc:write` scope — verify this is possible today (known gap), document the security implication.
- Verify the token format `bk_pat_<prefix>.<secret>` and that the SHA-256 hash covers the full string (not just `<secret>`).

---

### 2.7 Headless CI Bootstrap — Signup + PAT in One Call (HIGH)

**Why it matters.** The headless signup and signin flows (FEAT-002, FEAT-003) are the entry point for all CI automation. These endpoints use the Supabase Admin API to bypass email confirmation — no OTP email, no browser, one round trip. If 409 Conflict on duplicate email is not handled correctly, CI pipelines that run `POST /auth/signup` on every deploy silently fail instead of falling back to `POST /auth/signin`. The uniform 401 on bad credentials must not reveal whether the email exists (email enumeration risk).

**What commonly breaks.** The `POST /auth/signin` response includes both a Supabase session AND a fresh PAT in one payload — if the PAT mint fails after the session is created, the caller has a session they can't easily use. The 409 idempotency path is only safe if `POST /auth/signin` is also reliable.

**Dependencies.** Supabase Admin API `createUser(email_confirm: true)` → `mintPat()` → 201 `{ user, pat }` for new users; `POST /auth/signin` → `signInWithPassword()` + `mintPat()` → 200 `{ user, session, pat }` for returning users.

**What an experienced QA would check.**

- First-time signup with a new email → 201 with user + PAT; verify user exists in Supabase Auth with email confirmed, no confirmation email sent.
- Retry signup with same email → 409; response guides caller toward `POST /auth/signin`.
- Sign in with correct credentials → 200 with session + fresh PAT; verify the PAT works for Bearer calls.
- Sign in with wrong password → 401; verify no email-existence information in error message.
- Signup with no `pat_name` or `pat_scopes` → PAT omitted from response, not null-valued.

---

### 2.8 Viewer Write-Block via RLS (HIGH)

**Why it matters.** The ATC editor UI renders the full editing interface for all roles — there is no viewer-specific read-only mode. The only thing standing between a viewer and overwriting an ATC is the Supabase RLS policy `atcs_member_plus`. This policy has never been independently verified. If it's missing, incorrectly scoped, or has a bypass, viewers can silently mutate the test library — which breaks the RBAC model that stakeholders and PMs rely on.

**What commonly breaks.** Server Actions in Next.js don't automatically inherit the session context the same way route handlers do — it's possible for `saveAtcAction` to use a supabase client that is not correctly scoped to the caller's role. The RLS policy must match on both `workspace_id` and `role in ('member','admin','owner')`.

**Dependencies.** `saveAtcAction` → `createClient()` (server-side, inherits session) → `bunkai_save_atc` RPC (SECURITY INVOKER) → RLS `atcs_member_plus` policy on `atcs` table.

**What an experienced QA would check.**

- Authenticate as a `viewer` and call `saveAtcAction` directly (bypassing the UI's `canSave` guard) → RLS must block the write and return a meaningful error.
- Confirm the supabase client used inside the server action is the cookie-session client, not the service_role admin client.
- Verify `viewer` cannot insert into `atc_steps`, `atc_assertions`, or `atc_acceptance_criteria` directly either.
- Test the same negative case for an `atc:write` Bearer PAT issued by a viewer.

---

## Section 3 — State Machines That Matter

### 3.1 AtcStatus — `unrun → running → pass / fail / blocked / skipped`

**Why transitions matter.** The status of an ATC is the primary signal a QA lead reads to assess release readiness. A status of `pass` means someone ran this case and it passed; `blocked` means there's an external dependency stopping execution; `fail` means the feature broke. Getting stuck in `running` — because a run started but never completed — makes an ATC look "in progress" indefinitely, poisoning coverage dashboards.

**Most breakage-prone transitions.** The mechanism that drives ALL of these transitions does not yet exist in the codebase (`runs` entity missing). When it ships, the highest-risk transitions are `running → pass` and `running → fail` — they're the ones that record outcome, and they need to be atomic with any run-step data being written. The `pass/fail/blocked/skipped → running` re-run paths need to correctly snapshot prior state rather than appending to it.

**Terminal / forbidden states to guard.** `unrun → pass` (skipping `running`) must be blocked — an ATC cannot be marked pass without an execution record. Similarly, `unrun → fail` without a run is an integrity violation. The save-path must validate the prior state before allowing a transition.

**How corruption would be detected.** Today, not at all. When runs ship, a stuck `running` state (run started, never completed, no timeout or abort mechanism) would appear in the UI as a permanently spinning indicator with no resolution path. You'll want a test for run abandonment and abort paths (BK-36 covers abort).

---

### 3.2 MemberStatus — `invited → active → suspended`

**Why transitions matter.** A suspended member must have zero data access — every RLS policy checks `status = 'active'`. If the suspension is delayed or if the check is on the wrong column, a suspended member continues reading workspace data. Accidental suspension of the last admin can strand a workspace with no management path.

**Most breakage-prone transitions.** `active → suspended` and its reverse are writes to `workspace_members.status` gated by admin/owner RLS. The boundary case: what happens when an admin suspends themselves? They're still admin at the time of the write, so the RLS check passes — and the workspace has one fewer manager immediately after.

**Terminal / forbidden states to guard.** The `invited` state should not be settable directly via PATCH — it should only be reachable via the invite flow. A member row in `invited` status for longer than 7 days (invite expiry) accumulates as dead data with no cleanup mechanism today.

**How corruption would be detected.** A suspended member who can still read data would only be noticed by probing the API with known-suspended credentials. There's no alert or audit trail surface for "suspended user attempted access."

---

### 3.3 WorkspaceInvite — `pending → accepted / expired / revoked`

**Why transitions matter.** This is the only onboarding mechanism in Bunkai. Token rotation (`FEAT-013`) resets all three terminal states back to `pending` — it's the only recovery path from an expired or undelivered invite.

**Most breakage-prone transitions.** `pending → accepted` when the invitee uses an email that doesn't match — the 403 must fire before any `workspace_members` upsert. The `expired` state is passive (no cron, no DB trigger) — it's computed on-the-fly from `expires_at`. A bug in the `derivedStatus()` computation in the UI could show `pending` for a token that's actually expired, misleading admins.

**Terminal / forbidden states to guard.** An already-`accepted` invite must not be re-accepted. A `revoked` invite must not be accepted. Neither state should be reachable by an unauthenticated user.

**How corruption would be detected.** Duplicate `workspace_members` rows for the same `(workspace_id, user_id)` pair would indicate a re-acceptance bug — the UPSERT should prevent this, but it's worth verifying with a DB-level query after a duplicate acceptance attempt.

---

### 3.4 WorkspacePlan — `community → cloud → enterprise`

**Why transitions matter.** The plan field gates future feature access and billing. Today, upgrade/downgrade logic is not implemented, and the PATCH endpoint for workspaces only updates `name`. A plan value set incorrectly — either too high (granting premium features to a free-tier account) or too low (cutting off a paying customer) — has direct business impact.

**Most breakage-prone transitions.** Downgrade paths (`enterprise → cloud → community`) are typically the ones that break in SaaS: they require revoking access to premium features, and that logic usually lags the billing system. Flag this when the billing integration ships.

**Terminal / forbidden states to guard.** Plan changes without the owner's authorization must be blocked — verify PATCH `/workspaces/{id}` respects the `workspaces_update_owner` RLS policy. A plan outside the enum would be caught by the DB CHECK constraint, but a technically-valid incorrect plan has no automated detection.

**How corruption would be detected.** Only by a billing reconciliation process — there's no in-app plan-integrity alert.

---

## Section 4 — Silent Killers (Automated Processes)

### 4.1 `bunkai_atcs_refresh_tsv` — Fulltext Search Trigger

**What it does.** On every INSERT or UPDATE of `atcs.title` or `atcs.tags`, this DB trigger rebuilds the `tsv` tsvector column used by the GIN index for fulltext search. Without it, search results go stale: an ATC whose title was updated still appears under the old title in search.

**What breaks if it fails.** If the trigger is dropped, disabled, or fails to fire, the GIN index falls out of sync silently. Users searching for an ATC by its current title get no results; users searching for an old keyword that was removed still get hits. There is no visible error — just wrong search results.

**How failure is detected today.** Not at all. There's no alerting on trigger failures, no `tsv` freshness check in the health endpoint, and no UI indication that search is operating on stale data.

**Recommended QA strategy.** After each ATC title update, immediately search for the new title and verify the ATC appears. Add a synthetic regression check: update an ATC's title, immediately search for the new title, assert the ATC appears. Query the `tsv` column directly to confirm it reflects the updated content.

---

### 4.2 Invite Expiry — Passive, No Cron

**What it does.** Workspace invites expire 7 days after issuance (`expires_at = now() + interval '7 days'`). Unlike an active expiry (a cron that deletes or flags rows), Bunkai checks expiry passively at the moment of acceptance in FR-003. There is no background process and no notification to the admin.

**What breaks if it fails.** Expired invite rows accumulate indefinitely. The `derivedStatus()` function in the Members UI computes status from `expires_at` at read time — if this computation is wrong, admins see `pending` for expired invites and waste time troubleshooting unreachable links. If the `expires_at < now()` check is accidentally bypassed in the accept route, expired tokens grant membership.

**How failure is detected today.** Admin checks the Members page, sees an invite that should have expired, attempts to resend, and observes unexpected behavior.

**Recommended QA strategy.** Test acceptance with a token whose `expires_at` is in the past (set directly in the test seed). Verify `POST /api/v1/invites/accept` returns 409 with a distinct expiry message. Verify the Members UI shows `expired` (not `pending`) for this invite row.

---

### 4.3 Magic Link Audit Trail — `consumed_at` Never Stamped

**What it does.** When a user clicks a magic link, `/auth/callback` exchanges the PKCE code for a session. Bunkai's `magic_link_tokens` table records OTP issuances with a `consumed_at` column intended to mark single-use consumption. Per the feature map, `consumed_at` is never stamped. BR-040 claims a replay guard exists, but that guard is entirely internal to Supabase Auth.

**What breaks if it fails.** If Supabase Auth's internal replay guard has a regression, Bunkai has no independent layer to catch OTP reuse. The audit table records issuance but cannot confirm single-use consumption.

**How failure is detected today.** Not detectably at the application layer. The only indication would be a second session created for the same OTP.

**Recommended QA strategy.** Capture a magic link URL before clicking it. Click it once (session established). Click it again. Verify the second click returns an error from Supabase Auth's replay guard. Document that `consumed_at` remains null — this is a known architectural gap that limits Bunkai's independent audit capability.

---

### 4.4 PAT `last_used_at` — Fire-and-Forget Update

**What it does.** When a valid Bearer PAT is used, `requireBearerToken()` middleware updates `access_tokens.last_used_at` as a fire-and-forget operation. This supports audit trail and token lifecycle management.

**What breaks if it fails.** If the update silently fails (DB write error swallowed), `last_used_at` stays stale. An admin reviewing tokens may revoke an "unused" token that is actively used by a CI pipeline, causing an unexpected 401 in production.

**How failure is detected today.** Not at all — a consistently stale `last_used_at` might eventually be noticed by an observant admin, but there's no automated detection.

**Recommended QA strategy.** After a successful Bearer API call, query `access_tokens` directly via DBHub MCP and verify `last_used_at` was updated within the last minute. Include this as a DB-level assertion in integration tests for the Bearer middleware.

---

### 4.5 ISR Cache Revalidation — `revalidatePath()` After ATC Save

**What it does.** After `bunkai_save_atc` RPC succeeds, `saveAtcAction` calls `revalidatePath()` for both the ATC editor path and the project tree path. This tells Vercel's ISR cache to regenerate those pages, keeping the sidebar and ATC table current.

**What breaks if it fails.** The project tree shows stale data: an ATC whose title was updated still appears under the old title in the sidebar until natural ISR expiry. Users see the save succeed (toast notification fires) but the UI doesn't reflect the change — the classic "it didn't save" support ticket that's actually a cache invalidation bug.

**How failure is detected today.** The save appears to succeed, but the visible data doesn't update immediately after the page is refreshed.

**Recommended QA strategy.** After each ATC save, immediately refresh the project tree sidebar and verify the updated title/status appears. Include a post-save page load in the E2E test, not just a save action assertion.

---

## Section 5 — External Integrations: Failure Points

### 5.1 Supabase Auth

**Which flow stops if it's down.** Everything. Authentication (magic link + password), session refresh on every middleware call, and the workspace bootstrap RPC all depend on Supabase. A full Supabase Auth outage is a total system outage for Bunkai.

**Critical timeouts.** The PKCE code exchange at `/auth/callback` must complete before the OTP token expires. Supabase's default OTP validity window is short — a user who clicks the link after a delay may hit an expired code error with no retry path.

**Acceptable degradation.** None for authenticated flows. New logins cannot be established and all authenticated endpoints fail with 401.

**Known quirks.** The env var inconsistency flagged in discovery (`NEXT_PUBLIC_SUPABASE_ANON_KEY` vs. `SUPABASE_PUBLISHABLE_KEY`) needs to be resolved — the wrong key in `lib/env.ts` means the client is initialized with undefined credentials, producing cryptic Supabase errors rather than a clear misconfiguration message.

---

### 5.2 Resend (OTP Email Delivery)

**Which flow stops if it's down.** Magic link authentication (FEAT-001, FR-006) — the entire browser login flow. Headless clients using `POST /auth/signup` or `POST /auth/signin` are unaffected.

**Critical timeouts.** Email delivery latency directly impacts the user's experience of the login flow. If Resend delivery is delayed, the OTP may expire before the user clicks the link. There's no retry mechanism in the magic-link endpoint.

**Acceptable degradation.** Headless/CI flows continue to work (password auth, PAT reuse). Browser users are fully blocked until Resend recovers.

**Known quirks.** Resend is wired via Supabase Auth SMTP relay configuration, not called directly via the Resend SDK. Bunkai cannot inspect delivery status, detect bounces, or implement send-side retry logic. The invite flow has `RESEND_API_KEY` configured but NOT wired — invite URLs are console-logged only (FEAT-011 MVP gap).

---

### 5.3 Atlassian / Jira

**Which flow stops if it's down.** No production flow is affected today — the Jira import endpoint (`POST /api/v1/jira/import`) is planned but not implemented (FEAT-031, BK-17). `ATLASSIAN_API_TOKEN` is used only by QA tooling.

**Critical timeouts.** When the import ships, Jira JQL queries can return large result sets. Atlassian's API rate limits (typically 300 requests/minute per key) could cause partial imports — leaving `user_stories` partially populated with no clear error.

**Acceptable degradation.** Full — app continues without Jira. User stories can be created manually once that UI ships. Jira is an import source, not a runtime dependency.

**Known quirks.** A burst of QA test runs that all hit the Atlassian API in parallel could saturate the API key. Atlassian rate limit responses (429) must be handled gracefully, not surfaced as cryptic DB errors.

---

### 5.4 n8n (Workflow Automation)

**Which flow stops if it's down.** None in the current app. `N8N_API_URL` and `N8N_API_KEY` are env vars with no wiring in app code — n8n is QA tooling layer only, invisible to end users.

**Known quirks.** If n8n is expected to drive automated ATC execution in the future (plausible given the `run:execute` PAT scope), it will become a runtime dependency. Worth flagging when the execution architecture is designed.

---

## Section 6 — Dependency Cascade Between Flows

The two critical chains in Bunkai are the human onboarding chain and the CI automation chain. Both collapse completely if their first link fails.

### Chain 1 — Human Onboarding to ATC Authoring

```
Magic Link Auth (Supabase + Resend)
  |
  +--> Session Cookie Set (/auth/callback)
         |
         +--> Workspace Bootstrap (SECURITY DEFINER RPC)
                |
                +--> Project / Module / UserStory hierarchy (no UI yet -- manual seed)
                       |
                       +--> AcceptanceCriterion created (no UI yet -- manual seed)
                              |
                              +--> ATC Editor loads (requires pre-existing atcId row)
                                     |
                                     +--> ATC Save (bunkai_save_atc RPC)
                                            |
                                            +--> ISR cache revalidation
                                                   |
                                                   +--> Project Tree + ATC Table updated

Failure at Auth          = nobody enters, nothing works
Failure at Bootstrap     = workspace exists without owner member row -- inaccessible
Failure at ATC Save RPC  = data lost silently if RPC rolls back without surfaced error
```

Note the dependency gap: Project, Module, UserStory, and AC creation have no implemented UI routes. Seeding the data required for ATC authoring is currently a manual API or DB operation — an onboarding gap that blocks end-to-end testing of the full chain from a clean state.

### Chain 2 — CI / Headless PAT Flow

```
POST /auth/signup (first time) or /auth/signin (re-entry)
  |
  +--> PAT Minted (scopes: atc:write, run:execute)
         |
         +--> Bearer calls to /api/v1/atcs, /api/v1/workspaces, etc.
                |
                +--> [PLANNED] POST /api/v1/runs
                       |
                       +--> AtcStatus transitions: unrun -> running -> pass/fail
                              |
                              +--> Run History + Reporting

Failure at signup/signin = CI pipeline cannot bootstrap credentials
Failure at PAT mint      = all Bearer calls fail with 401
Failure at revocation    = revoked token keeps working (security gap)
Failure at runs (future) = ATC statuses never update; coverage always shows 'unrun'
```

The CI chain has a pending link: `POST /api/v1/runs` and the `runs` entity do not exist yet. Until they ship, automated ATC execution is architecturally blocked — the `run:execute` PAT scope exists but there is nothing to call with it.

---

## Section 7 — Edge Cases Developers Commonly Forget

### Concurrency

**Two members editing the same ATC simultaneously.** Both open the ATC editor, both make changes to the same steps, and both click Save within seconds of each other. The `bunkai_save_atc` RPC uses an upsert — the last write wins. There's no optimistic locking, no version conflict error, and no warning that another user's changes were overwritten. The `atcs.version` column exists but no trigger was found to increment it on save.

Most at risk: ATC Save (FEAT-022) whenever two members are in the same project.

### Data Limits

**Monaco editor content size.** The step content (`atc_steps.content`) and assertion content (`atc_assertions.content`) are Postgres `text` columns with no length cap — but Next.js body parser limits and Supabase RPC payload limits are practical ceilings. A QA engineer who pastes a large JSON blob into the assertions YAML field may hit these limits in a way that produces a cryptic error rather than a clear validation message.

Most at risk: ATC Save (FEAT-022) with unusually large step or assertion content.

### Permission Boundaries at the Edge

**Viewer issuing a high-privilege PAT.** A viewer-role member can call `POST /api/v1/tokens` with `scopes: ["atc:write", "workspace:admin"]` and receive a valid token. That token may allow write operations that the same user cannot perform via the browser UI, potentially bypassing UI-layer guards without hitting RLS.

Most at risk: PAT Create (FEAT-016) + any write endpoint called with a viewer-issued PAT.

**Cross-workspace URL guessing.** The project route is `/projects/[projectSlug]` with no workspace context in the URL. A member of workspace A who knows a project slug in workspace B could navigate to that URL. RLS should return empty data, but the page may render a confusing "no data" state rather than a 404.

Most at risk: Project View (FEAT-020).

### Orphaned States

**ATC with no steps and no assertions.** The anchoring moat enforces `acIds.length >= 1`, but there's no minimum requirement for steps or assertions. A member can save an ATC with a title, story, and AC link, but completely empty step and assertion content. The ATC is technically valid per all current checks but is useless as a test case. Run execution (when it ships) may fail or behave oddly when encountering a zero-step ATC.

Most at risk: ATC Save (FEAT-022) + future Run Execution (FEAT-033).

**Module with orphaned `parent_module_id`.** The `modules` table supports self-referential nesting. If a parent module is deleted without cascading to children, child modules become orphans with an invalid FK. The DB behavior in this case is undefined in the current migrations.

Most at risk: future Module CRUD (FEAT-029, BK-9–11).

### Idempotency

**Double-submit on ATC Save.** If the user clicks Save twice in quick succession, two `saveAtcAction` calls hit the server in parallel. Both target the same `atcId` and the RPC upserts — so both should converge on the same final state. But if the second call arrives while the first transaction is mid-flight, a constraint violation on `atc_steps.position` (unique per ATC) is possible depending on the RPC's upsert strategy.

Most at risk: ATC Save (FEAT-022) under slow-network conditions.

**Workspace bootstrap called twice.** If the onboarding form is submitted twice (double-click, network retry), the second call hits `bunkai_bootstrap_workspace` again. The slug uniqueness constraint catches it and returns 23505 — but does the client show a clean "Slug taken" error or a raw 500?

Most at risk: Workspace Bootstrap (FEAT-007) on onboarding form.

### Timezone / Locale

**Invite expiry computed in server time vs. client display.** `expires_at` is stored as `timestamptz` (UTC-anchored). The `derivedStatus()` function in the Members UI derives status at read time. If it computes `new Date(expires_at) < new Date()` without timezone normalization, an invite that expired at midnight UTC may show as `pending` or `expired` depending on the user's local timezone offset.

Most at risk: Invite Listing (FEAT-012) and `derivedStatus()` in `members/page.tsx`.

**Run timestamps (future).** When `runs` ships, run start/end times need consistent UTC handling. A run that crosses midnight UTC could be split across two calendar days in reporting, affecting daily trend graphs.

---

## Section 8 — Pre-Release Checklist

Ordered CRITICAL first, then HIGH.

1. Verify magic link OTP is delivered within 60 seconds in staging and the `/auth/callback` exchange completes without error.
2. Verify workspace bootstrap creates both a `workspaces` row AND a `workspace_members` row (role=owner) atomically — query the DB to confirm, don't rely on the UI response.
3. Verify ATC Save persists all four entity types (`atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`) atomically — DB-level check after a full save.
4. Verify cross-workspace RLS: user A's session cannot read user B's workspace ATCs, members, or invites — run with two independent staging accounts.
5. Verify viewer role cannot save an ATC even when bypassing the UI's `canSave` guard — call `saveAtcAction` directly and confirm RLS blocks the write.
6. Verify invite accept correctly rejects: expired token, revoked token, already-accepted token, email-mismatch — each must return a distinct 409 or 403 with a clear message.
7. Verify PAT soft revoke takes effect on the very next Bearer request — no grace period, no cache delay.
8. Verify headless signup 409 idempotency: `POST /auth/signup` with an existing email returns 409; subsequent `POST /auth/signin` with correct credentials returns 200 with a valid PAT.
9. Verify `bunkai_atcs_refresh_tsv` trigger fires after ATC title update — search for the new title immediately after save and confirm the ATC appears.
10. Verify the open-redirect guard on magic-link `?next=`: `next=//evil.com` is rejected; `next=/projects` is accepted and the redirect completes correctly.
11. Verify `atcs.tsv` GIN index does not leak cross-workspace results — user A searches for a keyword known only in workspace B and gets zero results.
12. Verify invite token rotation invalidates the old token and activates the new one — old token returns 409, new token returns 200.
13. Verify PAT scope enforcement: a token with only `atc:read` scope cannot call a write endpoint, even if the underlying user has `member` role.
14. Verify `GET /api/v1/health` returns `{ ok: true }` and is publicly accessible without auth — use this as the staging smoke-test entry point before any functional testing begins.
15. Verify `POST /api/v1/me/active-workspace` refuses to switch to a workspace the caller is not an active member of — membership check must fire before the cookie is set.

---

## Section 9 — What Is NOT in This Plan

```markdown
- Flow diagrams and state-machine transition tables
    --> `.context/business/business-data-map.md`

- Feature catalog, CRUD matrix, third-party integrations inventory
    --> `.context/business/business-feature-map.md`

- API endpoint contracts, auth model, token flow diagrams
    --> `.context/business/business-api-map.md`

- TypeScript types (generated from OpenAPI)
    --> `api/openapi-types.ts` (run `bun run api:sync`)

- Detailed test case definitions, step-by-step TC scripts, traceability
    --> TMS (see `/test-documentation`)

- Sprint-level execution order, ATP/ATR per ticket
    --> `.context/PBI/epics/*/stories/*/` (see `/sprint-testing`)
```

---

## Section 10 — Discovery Gaps

The following gaps could not be grounded in verified evidence during the discovery pass that produced the three source maps. Each represents a risk to testing completeness.

**CRITICAL — `runs` / `run_steps` entity not in DB.**
AtcStatus has six defined values (`unrun`, `running`, `pass`, `fail`, `blocked`, `skipped`) but no route, server action, migration, or DB table exists to drive transitions. The entire run-execution model (BK-34, BK-35, BK-36, BK-39) is planned but unimplemented. Testing ATC status transitions, run history, defect filing from failing steps, and coverage reporting is fully blocked until this ships. The `run:execute` PAT scope exists in the DB CHECK constraint but cannot be exercised against any endpoint today.

**HIGH — ATC creation path missing.**
`saveAtcAction` takes `atcId` as an input parameter — it updates an existing row. No `POST /api/v1/atcs` endpoint, no "create new ATC" server action, and no confirmed handler for the "New ATC" toolbar button were found during discovery. The end-to-end flow from clicking "New ATC" to landing in the editor with a pre-populated `atcId` is unclear and blocks end-to-end ATC authoring tests.

**HIGH — `bunkai_save_atc` RPC internals unverified.**
Migration 0007 defines this RPC but its source was not read. Atomicity of the four-table upsert is assumed from the calling code. The behavior on RLS violation (viewer write attempt) — whether it returns a DB error code or silently returns empty — is unconfirmed. This RPC source must be read before writing ATC Save test cases.

**HIGH — Viewer write-block at app layer unconfirmed.**
`saveAtcAction` does not check `workspace_members.role`. The only write barrier for viewer-role members is the Supabase RLS policy `atcs_member_plus`. This policy's source (migration 0004) was referenced but not fully read. If the policy has a gap, viewers can overwrite ATCs without any error.

**HIGH — PAT scope gate absent (role escalation path).**
`POST /api/v1/tokens` is session-gated only. A `viewer` can request `atc:write` or `workspace:admin` scopes and receive a valid PAT. Whether this is intentional or a security gap requires team clarification before writing PAT security test cases.

**HIGH — No security headers configured.**
`next.config.ts` has no `headers()` function. CSP, HSTS, X-Frame-Options, and X-Content-Type-Options are absent. Vercel-level header configuration was not confirmed. This blocks all security-header test cases.

**HIGH — No rate limiting on any endpoint.**
All 19 API endpoints are unthrottled. The magic-link endpoint is the highest-exposure case (public, no auth, triggers external email). "Phase F" is referenced in a code comment as the planned rate-limiting milestone. Until that ships, rate-limit behavior cannot be tested.

**MEDIUM — Invite email delivery MVP gap.**
`POST /api/v1/workspaces/{id}/invites` returns `accept_url` once in the response with a "copy now" warning. Resend is configured (`RESEND_API_KEY`) but not wired to the invite flow. Cannot test email delivery for invites in the current codebase.

**MEDIUM — Magic link replay guard (Bunkai audit layer).**
`magic_link_tokens.consumed_at` is never stamped by the `/auth/callback` route. BR-040 claims replay guard exists — this is enforced only by Supabase Auth internally. Bunkai's `magic_link_tokens` table is an audit log, not a true replay guard. Cannot independently verify single-use enforcement at the Bunkai application layer.

**MEDIUM — Staging URL unknown.**
The staging environment URL is not populated in `.agents/project.yaml`. Staging-specific behavior (Vercel preview builds, environment-scoped Supabase config) cannot be validated until this is confirmed.

**LOW — No CI pipeline.**
No `.github/workflows/` directory was found. Regression testing is entirely manual. This gap grows with each new feature sprint.

**LOW — RLS policies not audited via DB tool.**
Cross-workspace isolation is the highest-risk area in the system, but the actual SQL of every RLS policy on every table was not read during the discovery pass. A full RLS audit via DBHub MCP is recommended before the first production release.
