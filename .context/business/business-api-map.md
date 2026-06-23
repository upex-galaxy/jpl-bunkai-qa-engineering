# Business API Map — Bunkai TMS

> Generated: 2026-06-23
> Command: `/business-api-map`
> Last verified against OpenAPI: 2026-06-23
> Source: `c:/Projects/UPEX/upex-bunkai-tms/` + context files

---

## 1. Executive Summary

Bunkai TMS exposes a REST API under `/api/v1/` that serves two consumer types: human QA teams working through a Next.js web UI, and headless API clients (CI pipelines, QA bots, and AI agents) calling the same endpoints directly. Every meaningful product operation — from onboarding a workspace to authoring an ATC and linking it to an Acceptance Criterion — flows through these handlers. The API is not a separate service; it is the backend half of a Next.js 15 BFF monolith deployed on Vercel.

Authentication is dual-mode. Browser users authenticate once via a magic-link OTP flow (or email + password), and Supabase SSR sets an httpOnly session cookie that rides on every subsequent request. Headless callers authenticate using a Personal Access Token (PAT) in the `Authorization: Bearer` header. Both paths resolve to the same `ApiAuthContext` object inside route handlers, with the key difference that cookie sessions carry no explicit scopes (RLS is the sole constraint) while Bearer tokens carry an explicit scope list (`atc:read`, `atc:write`, `run:execute`, `workspace:admin`) checked by the `requireScopeOrCookie()` helper.

Multi-tenant data isolation is enforced at the PostgreSQL layer via Row-Level Security on every table. Every RLS policy subqueries `workspace_members` for the caller's active membership and role before permitting any read or write. This architecture means a Supabase RLS misconfiguration is the most impactful single failure point in the system — a gap in any policy could expose cross-workspace data. The application layer adds business-rule validation (Zod schemas, anchoring moat) on top of RLS but never substitutes for it.

---

## 2. Permission & Auth Model

### Auth Tiers

| Tier | Who | How to acquire | Where enforced |
|------|-----|---------------|----------------|
| Public | Anonymous browser visitors | — | No auth check in `middleware.ts` for public prefixes (`/login`, `/auth`, `/api/v1/health`, etc.) |
| Session-Authenticated | Signed-in users | Magic link OTP (`POST /api/v1/auth/magic-link`) or email+pw (`POST /api/v1/auth/signin`) → Supabase sets httpOnly cookie | `middleware.ts` `getUser()` on every request; route handlers call `supabase.auth.getUser()` |
| PAT-Bearer | Headless API clients (CI, agents, bots) | `POST /api/v1/tokens` (requires active session) | `requireAuth()` in `lib/api/auth.ts` → `requireBearerToken()` in `lib/api/middleware/bearer.ts` |
| Role-gated | workspace members with role >= threshold | Assigned at invite acceptance or owner bootstrap | Supabase RLS on all tables + optional `requireScopeOrCookie()` at handler level |

### Role Hierarchy

```
owner   (bootstrapped only -- cannot be invited)
  |
admin   (highest invitable role)
  |
member  (default invite role -- can author ATCs)
  |
viewer  (read-only -- write-block via RLS, not enforced at app layer)
```

Source: `lib/types.ts` MemberRole enum; `workspace_invites` route caps invited role at `admin`.

### PAT Scopes

| Scope | Grants |
|-------|--------|
| `atc:read` | Read ATC data via Bearer |
| `atc:write` | Mutate ATCs via Bearer |
| `run:execute` | Trigger test execution (future `runs` entity) |
| `workspace:admin` | Workspace-level admin operations via Bearer |

Note: scope gate is Bearer-only. Cookie sessions bypass scope checks — the UI is the gate for browser users.

### Token Flow Diagrams

**Session Cookie flow:**
```
Client --> POST /api/v1/auth/magic-link { email }
        --> Supabase Auth dispatches OTP email (via Resend SMTP config)
        --> Client clicks link --> GET /auth/callback?code=<pkce_code>&next=<path>
        --> Supabase PKCE code exchange --> session cookie set (sb-<ref>-auth-token, httpOnly)
        --> middleware.ts getUser() validates + refreshes cookie on every subsequent request
        --> Route handlers: supabase.auth.getUser() or requireAuth() returns { userId, source: 'cookie' }
```

**PAT Bearer flow:**
```
Client (session) --> POST /api/v1/tokens { scopes, name, expires_in_days }
                 --> lib/api/pat.ts mintPat() generates bk_pat_<prefix>.<secret>
                 --> access_tokens row inserted (prefix + SHA-256 hash)
                 --> access_token_secrets row inserted (hash only -- QA roles cannot read)
                 --> Raw token returned ONCE with warning

Client (headless) --> Authorization: Bearer bk_pat_<prefix>.<secret>
                  --> requireBearerToken() extracts prefix (12 chars)
                  --> SELECT access_tokens WHERE token_prefix = <prefix> (indexed O(1))
                  --> Fetch SHA-256 hash from access_token_secrets (sibling table)
                  --> SHA-256(prefix+remainder) === stored_hash AND revoked_at IS NULL AND expires_at > now()
                  --> Returns BearerContext { userId, workspaceId, scopes, tokenId }
                  --> last_used_at updated fire-and-forget
```

**Active workspace cookie:**
```
Client --> POST /api/v1/me/active-workspace { workspace_id }
        --> RLS verifies caller is active member of target workspace
        --> bk_active_ws httpOnly cookie set to workspace_id
        --> GET /api/v1/me reads cookie --> returns active_workspace_id in response
        --> UI re-renders to target workspace context
```

**requireAuth() dual-mode dispatcher (lib/api/auth.ts):**
```
Route handler calls requireAuth(request)
  |
  +--> Authorization header present and starts with "Bearer "?
  |         YES --> requireBearerToken() --> BearerContext
  |                 returns { userId, source: 'bearer', scopes, tokenId, workspaceId }
  |
  +--> NO --> supabase.auth.getUser() via cookie
              user null? --> ApiError('unauthorized')
              user found --> returns { userId, source: 'cookie', scopes: [], tokenId: null }
```

---

## 3. Critical Business Journeys

### Journey 1 — Owner Onboarding (First Workspace)

A new user registers, authenticates, and creates the workspace that all future team work lives in. This is the prerequisite for every other journey.

```
Browser --> POST /api/v1/auth/magic-link { email }
        --> Supabase OTP email dispatched (Resend)
        --> /auth/callback (PKCE exchange) --> session cookie set
        --> middleware.ts redirects unauthenticated /onboarding to /login
        --> Authenticated browser --> POST /api/v1/workspaces { slug, name }
        --> supabase.rpc('bunkai_bootstrap_workspace') [SECURITY DEFINER]
        --> Atomic INSERT workspaces + INSERT workspace_members(role=owner, status=active)
        --> 201 { workspace } --> client redirects to /projects
```

1. Magic link is the primary auth entry point — Resend delivers the OTP email via Supabase Auth SMTP relay.
2. `bunkai_bootstrap_workspace` uses SECURITY DEFINER to break the chicken-and-egg RLS problem: the user cannot insert a workspace (RLS requires active membership) until the workspace exists.
3. Slug must pass regex `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$` and not be in the 18-word reserved list.
4. 409 Conflict on duplicate slug — no alternative suggestion offered (known UX gap).

**Endpoints**: `POST /api/v1/auth/magic-link`, `POST /api/v1/workspaces`
**Entities**: `auth.users`, `workspaces`, `workspace_members`
**Feature IDs**: FEAT-001, FEAT-006, FEAT-007

---

### Journey 2 — Headless / CI Bootstrap (Provisioning + PAT)

CI pipelines and QA agents need a non-interactive path to obtain a long-lived token. This journey provisions a user account and returns a PAT in one round trip, bypassing the magic-link email flow entirely.

```
CI --> POST /api/v1/auth/signup { email, password, pat_name, pat_scopes }
    --> Supabase Admin API createUser(email_confirm: true) [no email sent]
    --> mintPat() --> access_tokens + access_token_secrets rows
    --> 201 { user, pat: { token, scopes } }
    --> CI stores PAT for all subsequent Bearer calls

Re-entry (idempotent) --> POST /api/v1/auth/signin { email, password }
                       --> signInWithPassword() + mintPat() in one call
                       --> 200 { user, session, pat }
```

1. Uses Supabase Admin API to confirm the email immediately — no OTP loop, no email sent.
2. 409 Conflict on duplicate email is idempotency-safe: caller switches to `POST /api/v1/auth/signin`.
3. The minted PAT has all requested scopes — no role gate at issuance (known security gap: viewer could issue `atc:write`).
4. `POST /api/v1/auth/signin` returns both Supabase session tokens AND a fresh PAT in one call — designed for CLI re-entry.

**Endpoints**: `POST /api/v1/auth/signup`, `POST /api/v1/auth/signin`, `POST /api/v1/tokens`
**Entities**: `auth.users`, `access_tokens`, `access_token_secrets`
**Feature IDs**: FEAT-002, FEAT-003, FEAT-016

---

### Journey 3 — Invite a Teammate (Onboard New Member)

An admin or owner invites a colleague by email, assigns a role, and the invitee joins the workspace by redeeming a single-use token.

```
Admin --> POST /api/v1/workspaces/{id}/invites { email, role }
       --> RLS verifies caller is admin or owner [bunkai_is_workspace_admin()]
       --> generateInviteToken() --> raw token + SHA-256 hash
       --> workspace_invites row (expires in 7 days)
       --> workspace_invite_secrets row (hash only -- QA roles cannot read)
       --> Response: { invite, accept_url, warning: "Copy this URL now" }
       --> Admin shares accept_url out-of-band (no email delivery in MVP)

Invitee --> authenticates via magic link or password
        --> POST /api/v1/invites/accept { token }
        --> hashInviteToken(token) --> lookup in workspace_invite_secrets
        --> Validate: not revoked, not accepted, not expired, email == caller email
        --> UPSERT workspace_members { workspace_id, user_id, role, status: active }
        --> Stamp workspace_invites.accepted_at + accepted_by_user_id
        --> 200 { ok, workspace_id, role }
```

1. Invite token hash lives in `workspace_invite_secrets`, separated from `workspace_invites` so analytics roles cannot reconstruct the raw URL.
2. Email match enforced: `invite.email.toLowerCase() !== user.email.toLowerCase()` → 403 Forbidden. Blocks token sharing.
3. Revoked, expired, and already-accepted invites each return 409 Conflict with distinct messages.
4. No email delivery in MVP — accept URL returned once in API response. Known workflow gap (FEAT-011).

**Endpoints**: `POST /api/v1/workspaces/{id}/invites`, `POST /api/v1/invites/accept`
**Entities**: `workspace_invites`, `workspace_invite_secrets`, `workspace_members`
**Feature IDs**: FEAT-011, FEAT-015

---

### Journey 4 — Author and Save an ATC

A QA engineer authors an Acceptance Test Case — writing steps in Markdown and assertions in YAML — and anchors it to a User Story and one or more Acceptance Criteria before saving. This is the core value of Bunkai.

```
Member (browser) --> GET /projects/{slug}/atcs/{atcId} (Server Component load)
                 --> ATC Editor renders: Monaco (steps MD), Monaco (assertions YAML), AnchoringPanel
                 --> Member fills: title, layer (UI/API/Unit), tags
                 --> AnchoringPanel: search user stories --> select >= 1 ACs
                 --> Client submits --> saveAtcAction (Server Action) {
                       atcId, title, layer, tags,
                       userStoryId, stepsMarkdown, assertionsYaml, acIds
                     }
                 --> App validation: userStoryId not empty, acIds.length >= 1, title not blank
                 --> parseStepsMarkdown() --> AtcStep[]
                 --> parseAssertionsYaml() --> AtcAssertion[]
                 --> saveAtc() --> supabase.rpc('bunkai_save_atc') [SECURITY INVOKER]
                 --> PostgreSQL: atomic upsert atcs + atc_steps + atc_assertions + atc_acceptance_criteria
                 --> revalidatePath(atcId path + project tree path) --> ISR cache bust
                 --> { ok: true } --> toast notification in UI
```

1. `bunkai_save_atc` RPC is SECURITY INVOKER — it inherits the caller's RLS context. A viewer's save attempt is blocked at the DB layer by `atcs_member_plus` RLS policy. The server action itself does not check role.
2. The "anchoring moat" (BR-021) is the defining product constraint: zero AC links → save rejected. Every ATC is traceable to a product requirement.
3. The save is atomic: all sub-record upserts succeed or roll back together in the RPC.
4. ISR cache bust via `revalidatePath()` keeps the project tree current without a full page reload.

**Endpoints**: Server Action `saveAtcAction` → `bunkai_save_atc` RPC (not a REST endpoint)
**Entities**: `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`, `user_stories`, `acceptance_criteria`
**Feature IDs**: FEAT-022

---

### Journey 5 — PAT Lifecycle Management

A QA lead issues a long-lived Bearer token for a CI pipeline, then revokes it when the pipeline is decommissioned.

```
Lead (session) --> POST /api/v1/tokens { name, scopes: ['atc:read', 'run:execute'], expires_in_days: 90 }
               --> generateSecret(32 bytes) --> base64url string
               --> tokenPrefix = secret.slice(0, 12)
               --> hash = SHA-256(prefix + remainder)  [full secret hashed]
               --> INSERT access_tokens (prefix, scopes, expires_at)
               --> INSERT access_token_secrets (token_id, hash) [QA roles cannot read]
               --> 201 { token: "bk_pat_<prefix>.<remainder>", warning: "Store now" }

CI --> Bearer calls with stored token
    --> requireBearerToken() verifies SHA-256(prefix+remainder) === stored_hash

Lead --> DELETE /api/v1/tokens/{id}
      --> Sets access_tokens.revoked_at = now() (soft revoke, no hard DELETE)
      --> 204 No Content
      --> All subsequent Bearer calls with that token return 401
```

1. Token format `bk_pat_<prefix>.<secret>` uses the `bk_pat_` family prefix as a scannable marker for secret-scanning tools (GitHub, GitGuardian).
2. The prefix (first 12 chars of base64url secret) enables O(1) indexed DB lookup before the constant-time SHA-256 comparison.
3. Hash of `prefix + remainder` (the full secret) is stored. Earlier bug hashed only `remainder`, causing perpetual 401s. Fixed in `lib/api/middleware/bearer.ts`.
4. Soft revoke preserves the audit trail. No DELETE RLS policy exists on `access_tokens`.

**Endpoints**: `POST /api/v1/tokens`, `GET /api/v1/tokens`, `DELETE /api/v1/tokens/{id}`
**Entities**: `access_tokens`, `access_token_secrets`
**Feature IDs**: FEAT-016, FEAT-017, FEAT-018

---

### Journey 6 — Workspace Switching (Multi-Workspace Context)

A user who belongs to multiple workspaces switches their active context — all subsequent API calls and UI state reflect the new workspace.

```
User --> WorkspaceSwitcher UI --> POST /api/v1/me/active-workspace { workspace_id }
      --> RLS verifies caller is active member of target workspace
      --> bk_active_ws httpOnly cookie set to workspace_id
      --> GET /api/v1/me --> reads bk_active_ws cookie --> active_workspace_id in response
      --> UI re-renders to target workspace context
```

1. `bk_active_ws` is a separate httpOnly cookie from the Supabase session cookie — it carries workspace context only.
2. RLS membership check prevents switching to a workspace the user is not a member of.
3. Bearer PAT callers get `active_workspace_id` from the PAT's own `workspace_id` field instead of the cookie.
4. `GET /api/v1/me` is the identity introspection endpoint — returns user id, email, workspace list, active workspace, and auth source + scopes.

**Endpoints**: `POST /api/v1/me/active-workspace`, `GET /api/v1/me`
**Entities**: `workspace_members`, `workspaces`
**Feature IDs**: FEAT-010, FEAT-019

---

### Journey 7 — Test Execution (PLANNED — Not Yet Implemented)

This journey does not exist in the current codebase. The `runs` and `run_steps` entities have no DB tables, no route handlers, and no server actions. `AtcStatus` transitions (`unrun → running → pass/fail/blocked/skipped`) are defined in the type system but no mechanism to trigger them exists.

```
[PLANNED -- no code exists as of 2026-06-23]

Member --> POST /api/v1/runs { atc_ids[], test_plan_id? }
        --> INSERT runs row (status: running)
        --> Per-ATC: INSERT run_steps
        --> Member records outcome --> PATCH /api/v1/runs/{id}/steps/{stepId}
        --> Run completes --> AtcStatus transitioned for each ATC
```

Blocked until the `runs` / `run_steps` migration ships (FEAT-033, Jira BK-34, BK-35, BK-36, BK-39).

**Feature IDs**: FEAT-033 (PLANNED)

---

## 4. Architecture Behind the API

```
+----------------------------------------------+
|  Client (Browser / API consumer)             |
+------------------+---------------------------+
                   | HTTPS / TLS 1.3
+------------------v---------------------------+
|  Vercel Edge + Serverless                    |
|  middleware.ts runs at edge:                 |
|    getUser() refresh, protected-path guard   |
|    Matcher: all paths except _next/static,   |
|    _next/image, favicon, file extensions     |
+------------------+---------------------------+
                   | Next.js runtime
+------------------v---------------------------+
|  Next.js 15 App Router (Vercel serverless)   |
|  app/api/v1/**  Route Handlers               |
|    withApiHandler() wraps all routes         |
|    requireAuth() dual-mode (cookie/bearer)   |
|    Zod schema validation on every body       |
|  app/(app)/**   Server Components + Actions  |
|    saveAtcAction() --> bunkai_save_atc RPC   |
|    RSC data fetching via createClient()      |
+------------------+---------------------------+
                   | Supabase JS client (HTTPS)
+------------------v---------------------------+
|  Supabase (managed PostgreSQL)               |
|  Row-Level Security on all 12 tables         |
|  auth.users (identity store)                 |
|  SECURITY DEFINER RPCs:                      |
|    bunkai_bootstrap_workspace                |
|  SECURITY INVOKER RPCs:                      |
|    bunkai_save_atc                           |
|  Triggers:                                   |
|    bunkai_atcs_refresh_tsv (GIN tsvector)    |
|    bunkai_set_updated_at                     |
+------------------+---------------------------+
                   | External HTTP calls
+------------------v---------------------------+
|  Resend          | OTP email delivery        |
|  Atlassian API   | Jira import (planned)     |
+----------------------------------------------+
```

| Component | Role | Persistence | QA Risk |
|-----------|------|-------------|---------|
| Vercel Edge middleware | Session refresh + protected-route redirect | Stateless | Stale cookie bugs; no security headers configured |
| Next.js route handlers | Request routing, auth wire-up, Zod validation | Supabase | Auth bypass, 4xx/5xx, schema drift |
| `requireAuth()` | Dual-mode dispatcher — Bearer checked first, then cookie | Stateless | Stale cookie shadowed by explicit PAT is handled; inverse is not |
| Supabase Auth | Identity, OTP, session cookie issuance | auth.users | Session hijack; OTP replay guard is Supabase-internal (Bunkai audit layer does not enforce it) |
| Supabase RLS | Multi-tenant data isolation on all 12 tables | PostgreSQL | Cross-workspace data leak — highest-impact single failure point |
| `bunkai_bootstrap_workspace` RPC | Atomic workspace + owner bootstrap (SECURITY DEFINER) | PostgreSQL | Partial-write on error may leave orphaned rows |
| `bunkai_save_atc` RPC | Atomic ATC sub-record upsert (SECURITY INVOKER) | PostgreSQL | Viewer write attempt fails at DB layer if RLS policy is correct — not confirmed |
| `access_token_secrets` table | SHA-256 hashes separated from readable `access_tokens` | PostgreSQL | Hash exposure enables token forgery; QA role isolation required |
| `workspace_invite_secrets` table | Invite token hashes separated from `workspace_invites` | PostgreSQL | Same isolation pattern as PAT secrets |
| Resend | Magic-link OTP email delivery | External (outbound) | OTP delivery failure prevents login |
| Atlassian API | Jira story import via JQL (planned) | External (outbound) | Credential leak in env; rate limit from Atlassian |

---

## 5. External Integrations

| Service | Package / Env var | Trigger | Direction | Failure mode (user-visible) | Journeys affected |
|---------|------------------|---------|-----------|-----------------------------|--------------------|
| Supabase Auth | `@supabase/ssr`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` | All auth flows, all DB operations | Outbound sync (every request) | Session invalid -> 401 / redirect to /login; DB error -> 500 | All authenticated journeys |
| Resend | `RESEND_API_KEY` (env — wired via Supabase Auth SMTP relay, not direct SDK call) | Magic link OTP request | Outbound sync | Email not delivered -> user cannot complete login | Journey 1, all magic-link flows (FEAT-001) |
| Atlassian / Jira | `ATLASSIAN_API_TOKEN` (env — QA tooling only; no app code wires this yet) | `POST /api/v1/jira/import` (planned) | Outbound sync | `jira_unauthorized` -> import fails; rate limit -> partial import | FEAT-031 (PLANNED) |
| n8n | `N8N_API_URL`, `N8N_API_KEY` (env — no app code wires this) | Workflow automation (planned QA tooling) | Outbound | Not user-visible today | None (tooling layer only) |
| DBHub | `POSTGRES_URL`, `POSTGRES_USER` etc. | QA test validation (direct DB access) | Outbound | QA environment only -- not user-visible | Test setup / teardown only |

---

## 6. Cross-References

- **Entity schemas + state machines** → `.context/business/business-data-map.md`
- **Feature catalog, CRUD matrix, full endpoint list** → `.context/business/business-feature-map.md`
- **TypeScript types (generated)** → `api/openapi-types.ts` (run `bun run api:sync`)
- **OpenAPI spec** → `GET /api/openapi` (runtime) or `app/api/openapi/route.ts`
- **SRS functional specs + business rules** → `.context/SRS/functional-specs.md`
- **User personas + role permission matrix** → `.context/PRD/user-personas.md`
- **System architecture (C4, DB schema, security gaps)** → `.context/SRS/architecture.md`
- **Auth middleware source** → `c:/Projects/UPEX/upex-bunkai-tms/middleware.ts`
- **requireAuth + Bearer middleware** → `lib/api/auth.ts`, `lib/api/middleware/bearer.ts`

---

## 7. Discovery Gaps

| Gap | Severity | Detail |
|-----|----------|--------|
| `runs` / `run_steps` entities not in DB | CRITICAL | `AtcStatus` has 6 values (`unrun/running/pass/fail/blocked/skipped`) but no route, server action, or migration exists to transition them. Journey 7 (test execution) is fully blocked. Jira: BK-34, BK-35, BK-36, BK-39. |
| ATC creation path missing | HIGH | `saveAtcAction` takes `atcId` as input — it updates an existing row. No `POST /api/v1/atcs` endpoint found. How is the first `atcs` row created? The "New ATC" button in the toolbar has no confirmed handler. This blocks ATC authoring automation. |
| `bunkai_save_atc` RPC source not verified | HIGH | Migration 0007 defines this RPC but was not read during discovery. Atomicity of sub-record upserts (atc_steps, atc_assertions, atc_acceptance_criteria) is assumed from calling code. Does it use a single transaction? What does it return on RLS violation (viewer write attempt)? |
| Viewer write-block at app layer unconfirmed | HIGH | `saveAtcAction` does not check `workspace_members.role`. Write-block relies entirely on Supabase RLS policy `atcs_member_plus`. If that policy has a gap, viewers can overwrite ATCs without any app-layer rejection. RLS policy source was not read during this discovery pass. |
| PAT scope gate absent (role escalation) | HIGH | `POST /api/v1/tokens` is session-gated only, not role-gated. A `viewer` can request `atc:write` or `workspace:admin` scopes and bypass UI-level restrictions via Bearer. Intentional (scope = capability grant) or a security gap — requires team clarification. |
| No security headers | HIGH | `next.config.ts` has no `headers()` function. No CSP, HSTS, X-Frame-Options, or X-Content-Type-Options configured. Vercel-level header config not confirmed. |
| Rate limiting absent | HIGH | No rate-limit middleware exists (`middleware.ts` comment references "Phase F"). All 19 endpoints are unthrottled. Magic-link endpoint most exposed (anonymous, triggers external email). |
| Invite email delivery MVP gap | MEDIUM | `POST /api/v1/workspaces/{id}/invites` returns `accept_url` once with a "copy now" warning. `RESEND_API_KEY` is configured but not wired to the invite flow. URL is lost if API response is not captured. |
| Magic link replay guard (Bunkai layer) | MEDIUM | `magic_link_tokens.consumed_at` is never stamped by the `/auth/callback` route. BR-040 claims replay guard — this is enforced only by Supabase Auth internally. Bunkai's `magic_link_tokens` table is an audit log, not a true replay guard. |
| Resend direct SDK usage not confirmed | LOW | `RESEND_API_KEY` is in env but no direct `resend` npm SDK call confirmed in Bunkai route handlers. Magic link email delivery appears to route through Supabase Auth SMTP relay. Direct Resend integration for invite emails is a future gap. |
| Jira import endpoint not yet implemented | LOW | `user_stories.external_id` / `external_url` columns and `ATLASSIAN_API_TOKEN` env var exist, but `POST /api/v1/jira/import` is planned only (FEAT-031). No route handler found. |
| Workspace delete blocked at API layer | LOW | No `DELETE /api/v1/workspaces/{id}` route found. RLS policy `workspaces_delete_owner` exists in migration 0001. Intentional MVP deferral — team should confirm whether workspace deletion is planned and on what timeline. |
