# Business Feature Map — Bunkai TMS

> Generated: 2026-06-23
> Command: `/business-feature-map`
> Source: `c:/Projects/UPEX/upex-bunkai-tms/` — Next.js 15 App Router monolith
> Cross-referenced: `business-data-map.md`, `functional-specs.md`, `user-personas.md`, `epic-tree.md`

---

## Inventory Summary

| Category | Count | Status |
|----------|-------|--------|
| Core (Auth & Identity) | 6 | Mixed — 4 Stable, 1 In Dev, 1 Planned |
| Core (Workspace Tenancy) | 4 | Stable |
| Core (ATC Library) | 3 | Stable |
| Core (Project & Module Hierarchy) | 2 | Stable (DB) / Planned (UI) |
| Core (User Stories & ACs) | 2 | Stable (DB) / Planned (UI) |
| Secondary (API Access — PATs) | 3 | Stable |
| Secondary (Invite Management) | 4 | Stable |
| Secondary (Navigation & UX) | 3 | Stable |
| Infrastructure | 3 | Stable |
| Planned / WIP | 11 | In Development / Backlog |
| **Total** | **41** | |

---

## Feature Catalog

### Domain: Authentication & Identity

#### FEAT-001 — Magic Link Authentication (Passwordless)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-001 |
| **Status** | Stable |
| **FR** | FR-006 |
| **Endpoints** | `POST /api/v1/auth/magic-link` |
| **UI** | `app/(auth)/login/page.tsx`, `magic-link-form.tsx` |
| **Users** | All (public entry point) |
| **Dependencies** | Supabase Auth OTP, Resend (email delivery), `magic_link_tokens` audit table |
| **Evidence** | `app/api/v1/auth/magic-link/route.ts`, `supabase/migrations/0009_cross_cutting.sql` |

**Capabilities:**
- [x] Email OTP dispatch via Supabase `signInWithOtp`
- [x] `?next=` redirect parameter forwarded to magic-link callback
- [x] Auth callback code exchange at `/auth/callback`
- [x] Audit trail — issuance logged to `magic_link_tokens` + `magic_link_token_secrets` (best-effort, swallowed on failure)
- [x] Rate-limit error mapping (upstream 429 → `rate_limited`)
- [ ] Replay guard enforcement — `consumed_at` never stamped; audit table records issuance only

---

#### FEAT-002 — Password Sign-In + PAT Mint (Headless / CLI)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-002 |
| **Status** | Stable |
| **FR** | — (CLI bootstrap path) |
| **Endpoints** | `POST /api/v1/auth/signin` |
| **UI** | None (headless only) |
| **Users** | QA bots, CI pipelines, CLI users |
| **Dependencies** | Supabase Auth password flow, `mintPat()` helper, `access_token_secrets` |
| **Evidence** | `app/api/v1/auth/signin/route.ts` |

**Capabilities:**
- [x] Email + password sign-in via Supabase `signInWithPassword`
- [x] PAT minted in same call (`pat_name`, `pat_scopes`, `pat_expires_in_days` optional)
- [x] Session cookies set on response (browser-compatible)
- [x] Uniform 401 — no email-enumeration leak
- [ ] Role-based PAT scope restriction absent — any authenticated user can request any scope

---

#### FEAT-003 — Headless User Provisioning (QA Bot / CI Sign-Up)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-003 |
| **Status** | Stable |
| **FR** | — (CLI bootstrap path) |
| **Endpoints** | `POST /api/v1/auth/signup` |
| **UI** | None (headless only) |
| **Users** | QA automation, CI pipelines |
| **Dependencies** | Supabase Admin API (`createUser` with `email_confirm: true`), `mintPat()` |
| **Evidence** | `app/api/v1/auth/signup/route.ts` |

**Capabilities:**
- [x] Admin-provisioned user (email confirmed immediately, no confirmation email)
- [x] PAT minted in same call — user + token ready in one round trip
- [x] 409 Conflict on duplicate email (idempotency-safe: caller uses signin instead)
- [ ] No role pre-assignment — user has no workspace membership until explicitly invited

---

#### FEAT-004 — OAuth (GitHub / Google) — PLANNED

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-004 |
| **Status** | Planned |
| **FR** | (BK-3) |
| **Endpoints** | None yet |
| **UI** | Disabled buttons in `app/(auth)/login/page.tsx` (opacity-60, `disabled`, `title="OAuth ships next sprint"`) |
| **Users** | All |
| **Dependencies** | Supabase OAuth providers (GitHub, Google) |
| **Evidence** | `app/(auth)/login/page.tsx` lines 137–155 |

**Capabilities:**
- [ ] GitHub OAuth sign-in
- [ ] Google OAuth sign-in

---

#### FEAT-005 — Email + Password Login UI — IN DEVELOPMENT

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-005 |
| **Status** | In Development |
| **FR** | (BK-166) |
| **Endpoints** | `POST /api/v1/auth/signin` (backend exists) |
| **UI** | Login page form not yet built |
| **Users** | All |
| **Dependencies** | `app/(auth)/login/page.tsx` — UI form TBD |
| **Evidence** | Jira BK-166 `Ready For QA` |

**Capabilities:**
- [x] Backend endpoint implemented (`/auth/signin`)
- [ ] Login page UI form for email+password not yet wired

---

#### FEAT-006 — Auth Callback + Session Cookie

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-006 |
| **Status** | Stable |
| **FR** | FR-006 |
| **Endpoints** | `GET /auth/callback` |
| **UI** | `app/auth/callback/` (route handler, no visible UI) |
| **Users** | All |
| **Dependencies** | Supabase Auth PKCE code exchange |
| **Evidence** | Middleware `getUser()` on every request |

**Capabilities:**
- [x] PKCE code exchange → session cookie (`sb-<ref>-auth-token`)
- [x] Redirect to `?next=` path after successful login
- [x] Open-redirect guard: `next` must start with `/` and not `//`
- [x] `bk_active_ws` cookie used for active workspace context across requests

---

### Domain: Workspace Tenancy

#### FEAT-007 — Workspace Bootstrap (Create Workspace)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-007 |
| **Status** | Stable |
| **FR** | FR-001 |
| **Endpoints** | `POST /api/v1/workspaces` |
| **UI** | `app/(app)/onboarding/page.tsx`, `onboarding-form.tsx` |
| **Users** | Any authenticated user without an active workspace |
| **Dependencies** | `bunkai_bootstrap_workspace` SECURITY DEFINER RPC |
| **Evidence** | `app/api/v1/workspaces/route.ts`, `supabase/migrations/0006_bootstrap_workspace.sql` |

**Capabilities:**
- [x] Atomic workspace + owner member row creation (single transaction, SECURITY DEFINER)
- [x] Slug validation: `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$` (3–40 chars)
- [x] Reserved slug rejection (18 reserved words: admin, api, app, auth, docs, invites, login…)
- [x] 409 on duplicate slug (SQLSTATE 23505 mapped to `conflict`)
- [x] New workspace defaults to `community` plan
- [x] Auto-redirect to `/projects` if workspace already exists (short-circuit in page)
- [ ] No slug-suggestion on collision — error shown, no alternative offered
- [ ] No workspace name uniqueness enforcement (slug is the unique key)

---

#### FEAT-008 — Workspace Listing

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-008 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | `GET /api/v1/workspaces` |
| **UI** | `WorkspaceSwitcher` component, `app/(app)/projects/page.tsx` (redirect to first project) |
| **Users** | Any active member |
| **Dependencies** | Supabase RLS (cookie path) or admin client + workspace_members join (Bearer path) |
| **Evidence** | `app/api/v1/workspaces/route.ts` |

**Capabilities:**
- [x] Cookie session path — RLS-filtered SELECT
- [x] Bearer PAT path — explicit workspace_members join (no RLS bypass leak)
- [x] Ordered by `created_at` ascending

---

#### FEAT-009 — Workspace Read / Update

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-009 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | `GET /api/v1/workspaces/{id}`, `PATCH /api/v1/workspaces/{id}` |
| **UI** | None found — API only |
| **Users** | GET: any active member; PATCH: owner only (RLS) |
| **Dependencies** | Supabase RLS `workspaces_update_owner` |
| **Evidence** | `app/api/v1/workspaces/[id]/route.ts` |

**Capabilities:**
- [x] Read workspace by ID (any active member)
- [x] Update workspace name (owner only via RLS)
- [ ] Slug update not allowed (slug immutable after bootstrap)
- [ ] Plan upgrade/downgrade not wired to this endpoint

---

#### FEAT-010 — Active Workspace Switch

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-010 |
| **Status** | Stable |
| **FR** | FR-008 |
| **Endpoints** | `POST /api/v1/me/active-workspace` |
| **UI** | `WorkspaceSwitcher` component in `Topbar` |
| **Users** | Any active member |
| **Dependencies** | `bk_active_ws` httpOnly cookie, RLS membership check |
| **Evidence** | `app/api/v1/me/active-workspace/route.ts` |

**Capabilities:**
- [x] Sets `bk_active_ws` httpOnly cookie (persists across page loads)
- [x] Verifies caller is a member of target workspace before switching (RLS check)
- [x] Cookie persists active workspace for `GET /api/v1/me` introspection

---

### Domain: Team & Invite Management

#### FEAT-011 — Workspace Member Invite (Issue)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-011 |
| **Status** | Stable |
| **FR** | FR-002 |
| **Endpoints** | `POST /api/v1/workspaces/{id}/invites` |
| **UI** | `app/(app)/workspaces/[id]/members/page.tsx`, `MembersClient` |
| **Users** | admin, owner |
| **Dependencies** | `workspace_invites`, `workspace_invite_secrets` (service_role), `generateInviteToken()` |
| **Evidence** | `app/api/v1/workspaces/[id]/invites/route.ts` |

**Capabilities:**
- [x] Issues invite for `(email, role)` pair; role ∈ {viewer, member, admin}
- [x] Token hash stored in sibling table (`workspace_invite_secrets`) — QA roles cannot read
- [x] Raw token + accept URL returned once in API response with `warning`
- [x] Expires 7 days from issuance (automatic)
- [x] RLS gates insert to admin/owner
- [ ] No email delivery — invite URL logged to console only (MVP gap)
- [ ] No bulk invite — single `(email, role)` pair per API call

---

#### FEAT-012 — Workspace Invite Listing

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-012 |
| **Status** | Stable |
| **FR** | FR-002 |
| **Endpoints** | `GET /api/v1/workspaces/{id}/invites` |
| **UI** | `app/(app)/workspaces/[id]/members/page.tsx` |
| **Users** | admin, owner |
| **Dependencies** | `workspace_invites` |
| **Evidence** | `app/api/v1/workspaces/[id]/invites/route.ts` |

**Capabilities:**
- [x] Lists all invites for a workspace, ordered by `created_at` desc
- [x] Derived status computed: `pending | accepted | revoked | expired`
- [x] Includes `accepted_at`, `revoked_at`, `expires_at` for status derivation

---

#### FEAT-013 — Invite Token Rotate (Resend)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-013 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | `POST /api/v1/workspaces/{id}/invites/{inviteId}` |
| **UI** | `MembersClient` (resend action) |
| **Users** | admin, owner |
| **Dependencies** | `workspace_invite_secrets` (service_role upsert), `generateInviteToken()` |
| **Evidence** | `app/api/v1/workspaces/[id]/invites/[inviteId]/route.ts` |

**Capabilities:**
- [x] Generates a fresh token + new 7-day expiry
- [x] Clears `accepted_at` and `revoked_at` (makes invite redeemable again)
- [x] Rotates secret in `workspace_invite_secrets` via upsert
- [x] Returns new raw token + accept URL (shown once)
- [ ] No email delivery — new URL returned in API response only

---

#### FEAT-014 — Invite Revocation

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-014 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | `DELETE /api/v1/workspaces/{id}/invites/{inviteId}` |
| **UI** | `MembersClient` (revoke action) |
| **Users** | admin, owner |
| **Dependencies** | `workspace_invites.revoked_at` (soft revoke) |
| **Evidence** | `app/api/v1/workspaces/[id]/invites/[inviteId]/route.ts` |

**Capabilities:**
- [x] Soft-revokes invite by setting `revoked_at = now()`
- [x] Revoked invites cannot be accepted (FR-003 validation)
- [ ] No hard delete — row stays in audit trail

---

#### FEAT-015 — Invite Accept (Join Workspace)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-015 |
| **Status** | Stable |
| **FR** | FR-003 |
| **Endpoints** | `POST /api/v1/invites/accept` |
| **UI** | `app/invites/accept/page.tsx`, `accept-client.tsx` |
| **Users** | Authenticated invitee (any, public landing) |
| **Dependencies** | `workspace_invite_secrets` (admin lookup), `workspace_members` (upsert) |
| **Evidence** | `app/api/v1/invites/accept/route.ts` |

**Capabilities:**
- [x] Token hash lookup via `workspace_invite_secrets` (service_role)
- [x] Validates: not expired, not revoked, not already accepted
- [x] Email match: caller's auth email must equal invite email
- [x] Upserts `workspace_members` with `status=active`, `role=invite.role`
- [x] Stamps `accepted_at` + `accepted_by_user_id` on invite row
- [ ] No redirect on success — caller navigates manually

---

### Domain: API Access (Personal Access Tokens)

#### FEAT-016 — PAT Create

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-016 |
| **Status** | Stable |
| **FR** | FR-005 |
| **Endpoints** | `POST /api/v1/tokens` |
| **UI** | None yet (BK-88: `Ready For Dev`) |
| **Users** | Any authenticated user (session-gated, not role-gated) |
| **Dependencies** | `access_tokens`, `access_token_secrets` (service_role) |
| **Evidence** | `app/api/v1/tokens/route.ts` |

**Capabilities:**
- [x] Generates `bk_pat_<12-char-prefix>.<secret>` format
- [x] Stores SHA-256 hash in `access_token_secrets` (QA roles cannot read)
- [x] Supports optional `workspace_id` scope (global if null)
- [x] Supports optional `expires_in_days` (up to 365)
- [x] Scopes validated: `atc:read`, `atc:write`, `run:execute`, `workspace:admin`
- [x] Raw token returned once with `warning` message
- [ ] Role gate missing — a `viewer` can issue a PAT with `atc:write` scope (security gap)

---

#### FEAT-017 — PAT Listing

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-017 |
| **Status** | Stable |
| **FR** | FR-005 |
| **Endpoints** | `GET /api/v1/tokens` |
| **UI** | None yet (BK-88: `Ready For Dev`) |
| **Users** | Any authenticated user |
| **Dependencies** | `access_tokens` RLS (owner-only via `auth.uid() = user_id`) |
| **Evidence** | `app/api/v1/tokens/route.ts` |

**Capabilities:**
- [x] Lists caller's own tokens only (RLS enforced)
- [x] Returns: id, name, scopes, workspace_id, token_prefix, expires_at, revoked_at, last_used_at, created_at
- [x] No secret hash exposed in response

---

#### FEAT-018 — PAT Revocation (Soft)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-018 |
| **Status** | Stable |
| **FR** | FR-005 |
| **Endpoints** | `DELETE /api/v1/tokens/{id}` |
| **UI** | None yet (BK-88: `Ready For Dev`) |
| **Users** | Any authenticated user (own tokens only, RLS enforced) |
| **Dependencies** | `access_tokens.revoked_at` (soft revoke) |
| **Evidence** | `app/api/v1/tokens/[id]/route.ts` |

**Capabilities:**
- [x] Soft-revoke by setting `revoked_at = now()`
- [x] Only revokes if not already revoked (`is revoked_at null` check)
- [x] No hard delete — row stays for audit trail
- [x] Returns 204 No Content on success

---

### Domain: Identity Introspection

#### FEAT-019 — User Identity Introspection (`/me`)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-019 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | `GET /api/v1/me` |
| **UI** | None (API introspection) |
| **Users** | Any authenticated user (cookie or Bearer) |
| **Dependencies** | Cookie session or PAT Bearer auth; admin client for Bearer path |
| **Evidence** | `app/api/v1/me/route.ts` |

**Capabilities:**
- [x] Returns: `user.id`, `user.email`, `workspaces[]`, `active_workspace_id`, `auth.source`, `auth.scopes`
- [x] Cookie path: RLS-filtered workspace list; `bk_active_ws` cookie for active workspace
- [x] Bearer path: explicit workspace_members join; `auth.workspaceId` from PAT

---

### Domain: Project & Module Hierarchy

#### FEAT-020 — Project View & ATC Table

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-020 |
| **Status** | Stable (read) / Planned (create UI) |
| **FR** | (BK-8) |
| **Endpoints** | None (Server Component read only) |
| **UI** | `app/(app)/projects/[projectSlug]/page.tsx`, `AtcTable`, `Sidebar` |
| **Users** | Any active member |
| **Dependencies** | `projects`, `modules`, `user_stories`, `acceptance_criteria`, `atcs` |
| **Evidence** | `app/(app)/projects/[projectSlug]/page.tsx` |

**Capabilities:**
- [x] Project tree view: modules → stories → ATCs (hierarchical sidebar)
- [x] ATC table: all ATCs for a project with status, layer, module path
- [x] "New ATC" and "New Test" buttons visible in toolbar
- [ ] Project creation UI — ships in Phase E (comment in `projects/page.tsx`)
- [ ] Multi-workspace project disambiguation (current: single-workspace assumption)

---

#### FEAT-021 — Project Index / Redirect

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-021 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | None (Server Component redirect) |
| **UI** | `app/(app)/projects/page.tsx` |
| **Users** | Any authenticated user |
| **Dependencies** | `workspace_members`, `projects` |
| **Evidence** | `app/(app)/projects/page.tsx` |

**Capabilities:**
- [x] Redirect to `/onboarding` if no active workspace membership
- [x] Redirect to first project if projects exist (ordered by `created_at`)
- [x] Empty-state "No projects yet" placeholder if workspace exists but no projects

---

### Domain: ATC Library

#### FEAT-022 — ATC Editor (View + Save)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-022 |
| **Status** | Stable |
| **FR** | FR-004 |
| **Endpoints** | Server Action: `saveAtcAction` → `bunkai_save_atc` RPC |
| **UI** | `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx`, `AtcEditor.tsx`, `AnchoringPanel.tsx` |
| **Users** | member, admin, owner (write); viewer (read — edit controls not role-gated at UI layer) |
| **Dependencies** | `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`, `user_stories`, `acceptance_criteria`, Monaco editor |
| **Evidence** | `app/(app)/projects/[projectSlug]/atcs/[atcId]/actions.ts`, `page.tsx` |

**Capabilities:**
- [x] Title, layer (UI/API/Unit), tags editing
- [x] Step authoring in Markdown (Monaco editor, `parseStepsMarkdown()`)
- [x] Assertion authoring in YAML (Monaco editor, `parseAssertionsYaml()`)
- [x] Anchoring panel: story search + AC multi-select
- [x] Save validation: userStoryId required, `acIds.length >= 1`, title not blank
- [x] Atomic upsert: atcs + atc_steps + atc_assertions + atc_acceptance_criteria via RPC
- [x] ISR cache bust: `revalidatePath` for ATC editor + project tree paths
- [ ] Viewer role not blocked at UI layer — relies on Supabase RLS for write-block
- [ ] ATC creation path unclear — `atcId` is a required input; how is the first row created?

---

#### FEAT-023 — ATC Fulltext Search

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-023 |
| **Status** | Stable (DB layer) / UI wiring pending |
| **FR** | FR-007 |
| **Endpoints** | None (direct DB query) |
| **UI** | `CommandPalette` component visible (wiring not confirmed) |
| **Users** | Any active member |
| **Dependencies** | `atcs.tsv` GIN index, `bunkai_atcs_refresh_tsv()` trigger |
| **Evidence** | `supabase/migrations/0004_atcs.sql` |

**Capabilities:**
- [x] `tsvector` built from title + tags (English language dictionary)
- [x] Auto-refreshed by DB trigger on INSERT/UPDATE of title or tags
- [x] GIN index (`atcs_tsv_gin_idx`) for performant `@@` queries
- [x] RLS scopes results to workspace (BR-050 — no cross-workspace leaks)
- [ ] Search API endpoint not found — query mechanism from UI unclear
- [ ] Language hardcoded to `'english'` (multi-language gap)

---

#### FEAT-024 — Members & Invites Page

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-024 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | Server Component read + API mutations via `MembersClient` |
| **UI** | `app/(app)/workspaces/[id]/members/page.tsx`, `MembersClient` |
| **Users** | Any active member (read); admin, owner (mutations) |
| **Dependencies** | `workspace_members`, `workspace_invites` |
| **Evidence** | `app/(app)/workspaces/[id]/members/page.tsx` |

**Capabilities:**
- [x] List current members with role + status + `joined_at`
- [x] List invites with derived status (pending/accepted/revoked/expired)
- [x] Invite action (admin/owner only)
- [x] Resend/rotate invite token action
- [x] Revoke invite action

---

### Domain: Infrastructure & Observability

#### FEAT-025 — Health Check

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-025 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | `GET /api/v1/health` |
| **UI** | None |
| **Users** | Public (no auth required) |
| **Dependencies** | None |
| **Evidence** | `app/api/v1/health/route.ts` |

**Capabilities:**
- [x] Returns `{ ok: true, service: 'bunkai-tms', env, ts }`
- [x] Force-dynamic (no ISR caching)

---

#### FEAT-026 — API Discovery + Interactive Docs

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-026 |
| **Status** | Stable |
| **FR** | — |
| **Endpoints** | `GET /api/v1`, `GET /api/openapi`, `GET /api/docs` |
| **UI** | `app/api/docs/page.tsx` (Scalar UI) |
| **Users** | Public |
| **Dependencies** | `@scalar/api-reference-react`, `public/openapi.json` |
| **Evidence** | `app/api/v1/route.ts`, `app/api/docs/page.tsx` |

**Capabilities:**
- [x] `/api/v1` — version banner + spec/docs pointers
- [x] `/api/openapi` — OpenAPI spec JSON
- [x] `/api/docs` — Scalar interactive docs
- [x] CORS preflight (`OPTIONS`) on `/api/v1`

---

#### FEAT-027 — Feature Flags (DB Infrastructure Only)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-027 |
| **Status** | Infrastructure — not yet consumed by app code |
| **FR** | — |
| **Endpoints** | None |
| **UI** | None |
| **Users** | service_role / admin tooling |
| **Dependencies** | `feature_flags` table (migration 0009) |
| **Evidence** | `supabase/migrations/0009_cross_cutting.sql` |

**Capabilities:**
- [x] `feature_flags` table: global + workspace-scoped flags with `key`, `enabled`, `payload`
- [x] RLS: authenticated users read global flags; workspace members read workspace-scoped flags
- [x] No client write policies — service_role / migrations only
- [ ] No app code consuming feature flags yet

---

### Domain: Planned / WIP Features

#### FEAT-028 — Project Creation UI — PLANNED
- **Evidence**: `app/(app)/projects/page.tsx` comment "Project creation UI ships in Phase E"
- **Jira**: BK-8 (QA Approved — DB schema exists)

#### FEAT-029 — Module CRUD UI — PLANNED
- **Evidence**: `modules` table fully defined (migration 0002); no UI routes
- **Jira**: BK-9, BK-10, BK-11 (all QA Approved)

#### FEAT-030 — User Story & AC Management UI — PLANNED
- **Evidence**: `user_stories` + `acceptance_criteria` tables exist; no UI create routes
- **Jira**: BK-14 (Ready For QA), BK-15 (QA Approved)

#### FEAT-031 — Jira JQL Import — PLANNED
- **Evidence**: `user_stories.external_id` + `external_url` columns exist for traceability
- **Jira**: BK-17 (QA Approved)

#### FEAT-032 — Test Builder (Chain of ATCs into Tests) — PLANNED
- **Evidence**: No `tests` or `test_steps` table in DB; no routes
- **Jira**: BK-27, BK-28, BK-32, BK-33

#### FEAT-033 — Manual Test Execution & Runs — PLANNED
- **Evidence**: No `runs` or `run_steps` entity in DB; `AtcStatus` enum fields exist but no status-transition mechanism in any route or action
- **Jira**: BK-34, BK-35, BK-36, BK-39

#### FEAT-034 — Run History & Reporting — PLANNED
- **Evidence**: No `run_history` table; `idempotency_keys` + `activity_log` infrastructure exists
- **Jira**: BK-37 (Backlog), BK-38 (Ready For Dev)

#### FEAT-035 — Bugs & Defect Heatmap — PLANNED
- **Evidence**: No defect entity in DB; no routes
- **Jira**: BK-40 (Ready For Dev), BK-41, BK-42, BK-43 (Backlog)

#### FEAT-036 — Coverage & Traceability Views — PLANNED
- **Evidence**: No coverage tables; `atc_acceptance_criteria` M:N join is the seed
- **Jira**: BK-45, BK-46, BK-47, BK-48, BK-49, BK-50 (Estimation / Shift-Left QA)

#### FEAT-037 — Settings Hub + PAT Management UI — PLANNED
- **Evidence**: PAT API complete (FEAT-016–018); no settings page route found
- **Jira**: BK-87 (Ready For Dev), BK-88 (Ready For Dev)

#### FEAT-038 — Workspace Leave — PLANNED
- **Evidence**: `workspace_members` DELETE RLS policy exists; no UI route
- **Jira**: BK-90 (Ready For Dev)

---

## CRUD Matrix

| Entity | Create | Read | Update | Delete | Evidence |
|--------|--------|------|--------|--------|----------|
| Workspace | ✅ `POST /workspaces` | ✅ `GET /workspaces`, `GET /workspaces/{id}` | ⚠️ Name only — owner via RLS | ❌ No delete route found | `workspaces/route.ts`, `[id]/route.ts` |
| WorkspaceMember | ✅ Via invite accept | ✅ `/members` page | ⚠️ Via token rotate (role implied) | ⚠️ RLS allows DELETE; no UI route | `invites/accept/route.ts` |
| WorkspaceInvite | ✅ `POST /workspaces/{id}/invites` | ✅ `GET /workspaces/{id}/invites` | ✅ Token rotate `POST /invites/{inviteId}` | ✅ Soft revoke `DELETE /invites/{inviteId}` | `[id]/invites/route.ts` |
| Project | ⚠️ DB RLS allows; no UI | ✅ Server Component | ❌ No update route | ❌ No delete route | `migrations/0002`, `[projectSlug]/page.tsx` |
| Module | ⚠️ DB RLS allows; no UI | ✅ Server Component | ❌ No update route | ❌ No delete route | `migrations/0002`, `[projectSlug]/page.tsx` |
| UserStory | ⚠️ DB RLS allows; no UI | ✅ Server Component (ATC editor) | ❌ No update route | ❌ No delete route | `migrations/0003` |
| AcceptanceCriterion | ⚠️ DB RLS allows; no UI | ✅ Server Component (ATC editor) | ❌ No update route | ❌ No delete route | `migrations/0003` |
| Atc | ⚠️ Creation path unclear | ✅ `AtcEditorPage`, `AtcTable` | ✅ `saveAtcAction` → `bunkai_save_atc` RPC | ❌ No delete route found | `actions.ts`, `page.tsx` |
| AtcStep | ✅ Via `saveAtcAction` (upsert) | ✅ ATC editor page | ✅ Via `saveAtcAction` | ✅ Via `saveAtcAction` (atomic replace) | `actions.ts` |
| AtcAssertion | ✅ Via `saveAtcAction` (upsert) | ✅ ATC editor page | ✅ Via `saveAtcAction` | ✅ Via `saveAtcAction` (atomic replace) | `actions.ts` |
| AtcAcceptanceCriterion | ✅ Via `saveAtcAction` | ✅ ATC editor page | ✅ Via `saveAtcAction` | ✅ Via `saveAtcAction` | `actions.ts` |
| AccessToken (PAT) | ✅ `POST /tokens` | ✅ `GET /tokens` | ❌ Scopes immutable; no update | ✅ Soft revoke `DELETE /tokens/{id}` | `tokens/route.ts` |
| auth.users | ✅ `POST /auth/signup` (headless) | ✅ `GET /me` (partial — id + email) | ❌ No profile update route | ❌ No delete via API | `signup/route.ts`, `me/route.ts` |
| feature_flags | ❌ service_role only | ✅ DB (authenticated users see global) | ❌ service_role only | ❌ service_role only | `migrations/0009` |
| activity_log | ❌ service_role only | ✅ DB (workspace members via RLS) | ❌ | ❌ | `migrations/0009` |
| idempotency_keys | ⚠️ DB RLS allows self-insert; no route wires it | ✅ DB (owner-only) | ⚠️ DB RLS allows self-update | ❌ | `migrations/0009` |
| user_view_state | ⚠️ DB RLS allows; no API | ⚠️ DB RLS allows; no API | ⚠️ DB RLS allows; no API | ⚠️ DB RLS allows; no API | `migrations/0009` |

Legend: ✅ Full · ⚠️ Partial/conditional · ❌ Not available

---

## API Endpoint Inventory

### Auth / Identity

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/auth/magic-link` | Dispatch OTP email; optional `?next=` redirect | Public |
| POST | `/api/v1/auth/signup` | Provision user (email_confirm=true) + mint PAT in one call | Public (admin client) |
| POST | `/api/v1/auth/signin` | Email+password sign-in + mint fresh PAT | Public |
| GET | `/auth/callback` | PKCE code exchange → session cookie; redirects to `next` | Public (OTP callback) |
| GET | `/api/v1/me` | User identity, workspaces, active_workspace_id, auth.source + scopes | Cookie or Bearer |

### Workspaces

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/workspaces` | Create workspace (atomic bootstrap + owner enroll) | Cookie (session) |
| GET | `/api/v1/workspaces` | List caller's workspaces (RLS or admin join) | Cookie or Bearer |
| GET | `/api/v1/workspaces/{id}` | Read single workspace | Cookie (session) |
| PATCH | `/api/v1/workspaces/{id}` | Update workspace name (owner only) | Cookie (session) |
| POST | `/api/v1/me/active-workspace` | Switch active workspace — sets `bk_active_ws` cookie | Cookie (session) |

### Invites

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/workspaces/{id}/invites` | Issue invite for (email, role) | Cookie — admin/owner |
| GET | `/api/v1/workspaces/{id}/invites` | List workspace invites with derived status | Cookie — admin/owner |
| POST | `/api/v1/workspaces/{id}/invites/{inviteId}` | Rotate invite token (resend) | Cookie — admin/owner |
| DELETE | `/api/v1/workspaces/{id}/invites/{inviteId}` | Soft-revoke invite | Cookie — admin/owner |
| POST | `/api/v1/invites/accept` | Accept invite by raw token (email match enforced) | Cookie (session, authenticated) |

### Personal Access Tokens

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/tokens` | Mint PAT with scopes + optional workspace/expiry | Cookie (session) |
| GET | `/api/v1/tokens` | List caller's PATs — no secrets exposed | Cookie (session) |
| DELETE | `/api/v1/tokens/{id}` | Soft-revoke PAT | Cookie (session) |

### Infrastructure

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1` | Version banner + spec/docs pointers; CORS OPTIONS | Public |
| GET | `/api/v1/health` | Liveness probe | Public |
| GET | `/api/openapi` | OpenAPI spec JSON | Public |
| GET | `/api/docs` | Scalar interactive API documentation | Public |

**Total confirmed API endpoints: 19**

---

## UI Component Inventory

### Pages / Routes

| Route | Page | Purpose | Auth Gate |
|-------|------|---------|-----------|
| `/` | `app/page.tsx` | Root redirect → `/login` | Public |
| `/login` | `app/(auth)/login/page.tsx` | Magic-link form + disabled OAuth buttons | Public |
| `/auth/callback` | Route handler | OTP code exchange, session cookie | Public (OTP) |
| `/onboarding` | `app/(app)/onboarding/page.tsx` | Workspace creation form | Authenticated, no workspace |
| `/projects` | `app/(app)/projects/page.tsx` | Project index / redirect | Active member |
| `/projects/[slug]` | `app/(app)/projects/[slug]/page.tsx` | Project view: sidebar + ATC table | Active member |
| `/projects/[slug]/atcs/[id]` | `app/(app)/projects/[slug]/atcs/[id]/page.tsx` | ATC editor: steps, assertions, anchoring | Active member |
| `/workspaces/[id]/members` | `app/(app)/workspaces/[id]/members/page.tsx` | Members + invites management | Active member |
| `/invites/accept` | `app/invites/accept/page.tsx` | Invite acceptance landing (token in query param) | Authenticated invitee |
| `/api/docs` | `app/api/docs/page.tsx` | Scalar interactive docs | Public |
| `/qa` | `app/qa/page.tsx` | QA testability guide | Public (internal) |
| `/design-tokens` | `app/design-tokens/page.tsx` | Design system reference | Public (internal) |

### Forms & Client Actions

| Component | Page | Purpose | Entity |
|-----------|------|---------|--------|
| `MagicLinkForm` | `/login` | Email input → OTP send | auth.users |
| `OnboardingForm` | `/onboarding` | Workspace slug + name → create | Workspace |
| `AtcEditor` | `/projects/[slug]/atcs/[id]` | Title, layer, tags, steps (MD), assertions (YAML) + save | Atc, AtcStep, AtcAssertion |
| `AnchoringPanel` | Inside AtcEditor | Story search + AC multi-select | UserStory, AcceptanceCriterion |
| `MembersClient` | `/workspaces/[id]/members` | Invite form + member/invite list with actions | WorkspaceMember, WorkspaceInvite |
| `AcceptClient` | `/invites/accept` | Token redemption → `POST /invites/accept` | WorkspaceInvite, WorkspaceMember |

### Views / Dashboards

| Component | Page | Purpose |
|-----------|------|---------|
| `AtcTable` | `/projects/[slug]` | Tabular ATC list with status chips, layer, module path |
| `Sidebar` | `/projects/[slug]` | Hierarchical module/story/ATC tree navigator |
| `Topbar` | App shell | Breadcrumb + WorkspaceSwitcher + action buttons (New ATC, New Test) |
| `WorkspaceSwitcher` | App shell | Dropdown to switch active workspace (`POST /me/active-workspace`) |
| `CommandPalette` | App shell | Keyboard-driven search (`cmdk`; wiring to search endpoint pending) |

---

## Third-Party Integrations

| Service | Package | Purpose | Status | Features using it |
|---------|---------|---------|--------|-------------------|
| Supabase Auth | `@supabase/ssr` `@supabase/supabase-js` | Magic-link OTP, session cookies, PAT verification | Active | FEAT-001 through FEAT-019 (all auth-dependent) |
| Supabase PostgreSQL | `@supabase/supabase-js` | All persistent data with RLS enforcement | Active | All data features |
| Resend | `RESEND_API_KEY` (env) | Transactional email — magic links and invite emails | Configured, NOT wired to invite flow (MVP gap) | FEAT-001 (OTP), FEAT-011 (gap) |
| Monaco Editor | `@monaco-editor/react` | ATC step (Markdown) + assertion (YAML) authoring | Active | FEAT-022 |
| Scalar | `@scalar/api-reference-react` | Interactive API documentation | Active | FEAT-026 |
| Radix UI | `@radix-ui/react-*` | Dialog, DropdownMenu, Tabs, Tooltip primitives | Active | All UI components |
| TanStack Table | `@tanstack/react-table` | ATC table sorting and column management | Active | FEAT-020 |
| cmdk | `cmdk` | Command palette (`⌘K` / `Ctrl+K`) | Wired (partial — search endpoint TBD) | FEAT-023 (UI) |
| Sonner | `sonner` | Toast notifications (save feedback, errors) | Active | FEAT-022 |
| Zod | `zod` | Request body validation on all API routes | Active | All 19 API endpoints |
| Lucide React | `lucide-react` | Icon library | Active | UI throughout |
| n8n | `N8N_API_URL` / `N8N_API_KEY` (env) | Workflow automation (MCP integration, QA tooling) | Env configured, no app code | None (tooling only) |
| Jira / Atlassian | `ATLASSIAN_API_TOKEN` (env) | User story traceability (`external_id`, `external_url`) | DB columns exist; import UI planned | FEAT-031 (planned) |
| DBHub | `DBHUB_*` (env) | QA database access via MCP | QA tooling only | QA / testing |
| OpenAPI MCP | `API_BASE_URL`, `OPENAPI_SPEC_PATH` (env) | API exploration for QA agents | QA tooling only | QA / testing |
| Vercel | (platform) | Hosting + edge deployment | Active (inferred) | All production |

---

## Feature Flags & WIP

### Feature Flags

No runtime feature flags found. `lib/env.ts` defines no `FEATURE_*`, `ENABLE_*`, or `BETA_*` environment variables. The `feature_flags` table (migration 0009) exists as infrastructure but no app code reads it.

| Flag | Description | Default |
|------|-------------|---------|
| *(none configured)* | Table exists; no flags seeded or consumed | — |

Feature gating is currently done via disabled UI elements (OAuth buttons with `disabled` attribute) and code-comment milestones ("Phase E", "Phase F").

### Planned / WIP Features (from Code + Jira)

| Planned Feature | Evidence | Jira Status |
|----------------|----------|-------------|
| OAuth (GitHub / Google) | Disabled buttons in login page | BK-3: Ready For Dev |
| Email + password login UI | Backend ready; no form | BK-166: Ready For QA |
| Project creation UI | Code comment "Phase E" | BK-8: QA Approved |
| Module tree CRUD UI | DB exists; no UI routes | BK-9, 10, 11: QA Approved |
| User Story + AC management UI | DB exists; no UI routes | BK-14: Ready For QA; BK-15: QA Approved |
| Jira JQL Import | `external_id` column exists | BK-17: QA Approved |
| ATC creation flow | `saveAtcAction` needs pre-existing `atcId` | BK-18, 19: QA Approved |
| ATC Propagation | No code | BK-21: Ready For Dev |
| ATC Duplicate | No code | BK-23: Ready For QA |
| ATC Usage report ("Used in N tests") | No code | BK-22: Ready For QA |
| Test Builder (chain ATCs) | No `tests` entity | BK-27, 28, 32, 33: QA Approved |
| Manual Execution & Runs | No `runs` entity; no status transition | BK-34–36, 39: Estimation/Dev |
| Run History & Reporting | No run tables | BK-37 Backlog; BK-38 Dev |
| Defect Filing + Heatmap | No defect entity | BK-40–43: Backlog/Dev |
| Coverage & Traceability views | No coverage tables | BK-45–50: Estimation |
| Settings Hub + PAT Management UI | PAT API complete; no settings page | BK-87, 88: Ready For Dev |
| Workspace Leave UI | RLS allows; no UI route | BK-90: Ready For Dev |
| Rate-limit middleware | Code comment "Phase F adds real middleware" | Not in Jira yet |
| Idempotency key enforcement | Table exists; not wired in any route | Not in Jira yet |
| Activity log (read side API) | Table exists; no read endpoint | BK-49: Shift-Left QA |

---

## QA Relevance

### Feature Test Coverage Matrix

| Feature ID | Feature | Unit | Integration | E2E | Priority |
|------------|---------|------|-------------|-----|----------|
| FEAT-001 | Magic Link Auth | ❌ | ⚠️ Endpoint only | ⚠️ UI: send + confirm state | P0 |
| FEAT-002 | Password Sign-In + PAT Mint | ❌ | ❌ | ❌ | P0 |
| FEAT-003 | Headless User Provisioning | ❌ | ❌ | ❌ | P0 |
| FEAT-004 | OAuth GitHub/Google | N/A | N/A | N/A | Planned |
| FEAT-005 | Email+Password Login UI | N/A | N/A | N/A | In Dev |
| FEAT-006 | Auth Callback + Session | ❌ | ❌ | ❌ | P0 |
| FEAT-007 | Workspace Bootstrap | ❌ | ❌ | ❌ | P0 |
| FEAT-008 | Workspace Listing | ❌ | ❌ | ❌ | P1 |
| FEAT-009 | Workspace Read/Update | ❌ | ❌ | ❌ | P1 |
| FEAT-010 | Active Workspace Switch | ❌ | ❌ | ❌ | P2 |
| FEAT-011 | Invite Issue | ❌ | ❌ | ❌ | P0 |
| FEAT-012 | Invite Listing | ❌ | ❌ | ❌ | P1 |
| FEAT-013 | Invite Token Rotate | ❌ | ❌ | ❌ | P1 |
| FEAT-014 | Invite Revocation | ❌ | ❌ | ❌ | P1 |
| FEAT-015 | Invite Accept | ❌ | ❌ | ❌ | P0 |
| FEAT-016 | PAT Create | ❌ | ❌ | ❌ | P1 |
| FEAT-017 | PAT Listing | ❌ | ❌ | ❌ | P2 |
| FEAT-018 | PAT Revocation | ❌ | ❌ | ❌ | P1 |
| FEAT-019 | /me Introspection | ❌ | ❌ | ❌ | P1 |
| FEAT-020 | Project View & ATC Table | ❌ | ❌ | ❌ | P1 |
| FEAT-021 | Project Index/Redirect | ❌ | ❌ | ❌ | P2 |
| FEAT-022 | ATC Editor (Save) | ❌ | ❌ | ❌ | P0 |
| FEAT-023 | ATC Fulltext Search | ❌ | ❌ | ❌ | P1 |
| FEAT-024 | Members & Invites Page | ❌ | ❌ | ❌ | P1 |
| FEAT-025 | Health Check | ❌ | ❌ | ❌ | P3 |
| FEAT-026 | API Docs | ❌ | ❌ | ❌ | P3 |
| FEAT-027 | Feature Flags (DB) | ❌ | ❌ | N/A | P3 |

Legend: ✅ Covered · ⚠️ Partial · ❌ Not covered · N/A Not applicable

### High-Risk Features

| Feature | Risk Level | Reason |
|---------|-----------|--------|
| FEAT-001 Magic Link Auth | HIGH | Entry gate — all other features blocked if auth broken |
| FEAT-007 Workspace Bootstrap | HIGH | All data scoped to workspaces; atomic RPC failure leaves partial state |
| FEAT-015 Invite Accept | HIGH | Token-based flow — replay, expiry, and email-mismatch bugs are security holes |
| FEAT-022 ATC Save | HIGH | Core value — anchoring moat + atomic sub-record upsert; RPC failure silently loses work |
| FEAT-016 PAT Create | HIGH | Bearer auth for CI/API; role-gate gap allows viewer to mint `atc:write` scope |
| FEAT-002 Password Sign-In | HIGH | Primary CI bootstrap path; uniform 401 must not leak email existence |
| FEAT-003 Headless Sign-Up | HIGH | One-time user provisioning; 409 idempotency must be reliable |
| FEAT-011 Invite Issue | MEDIUM | No email delivery — URL lost if API response not captured; no resend in current UI |
| FEAT-018 PAT Revocation | MEDIUM | Soft-revoke must block Bearer auth immediately; Bearer middleware must check `revoked_at` |
| FEAT-023 ATC Search | MEDIUM | GIN index + RLS isolation — cross-workspace data leak would be a critical breach |

---

## CRUD vs Data Map Cross-Reference

### Entities in `business-data-map.md` with no corresponding implemented feature

| Entity | Data Map Status | Feature Coverage | Risk |
|--------|----------------|------------------|------|
| `runs` | CRITICAL GAP (noted in data map) | No entity in DB; no routes; `AtcStatus` exists but no transition mechanism | All status transitions blocked — cannot test status changes |
| `run_steps` | Implied by AtcStep structure | No DB table; no routes | Cannot record per-step execution results |
| `user_view_state` | DB exists (migration 0009) | No API endpoint | View persistence not surfaced to app |
| `idempotency_keys` | DB exists (migration 0009) | Table not wired in any route handler | POST replay protection not active |
| `activity_log` | DB exists (migration 0009) | No read API; service_role write only | Audit trail exists but inaccessible via API |
| `feature_flags` | DB exists (migration 0009) | No app code reads flags yet | Feature gating infrastructure unused |
| `magic_link_tokens` | DB exists | Issuance recorded best-effort; `consumed_at` never stamped | Replay guard not enforced |

### Features with persistence but no implemented UI create/edit flow

| Entity | DB | RLS (write) | API Endpoint | UI Route |
|--------|-----|------------|-------------|----------|
| Project | ✅ | ✅ member+ | ❌ | ❌ Phase E |
| Module | ✅ | ✅ member+ | ❌ | ❌ planned |
| UserStory | ✅ | ✅ member+ | ❌ | ❌ planned |
| AcceptanceCriterion | ✅ | ✅ member+ | ❌ | ❌ planned |
| Atc (creation) | ✅ | ✅ member+ | ❌ | ⚠️ editor exists but requires pre-existing row |

---

## Discovery Gaps

| Gap | Severity | Team Question |
|-----|----------|---------------|
| ATC creation flow | HIGH | `saveAtcAction` takes `atcId` as INPUT — it updates an existing row. How is the first `atcs` row created? Is there a "new ATC" endpoint that pre-inserts an empty row and redirects to the editor? |
| `bunkai_save_atc` RPC internals | HIGH | Migration 0007 defines this RPC but was not read in this discovery pass. Does it upsert atc_steps + atc_assertions + atc_acceptance_criteria atomically? What does it return on RLS violation (viewer write attempt)? |
| ATC status transition mechanism | CRITICAL | `AtcStatus` has 6 values (unrun/running/pass/fail/blocked/skipped) but NO route or action transitions `atcs.status`. The `runs` entity that drives transitions does not exist in the DB. What is the planned mechanism and timeline? |
| Viewer write-block at app layer | HIGH | `saveAtcAction` does not check caller role — relies entirely on Supabase RLS on `atcs`. Has the RLS policy `atcs_member_plus` been verified to block viewer writes in practice? |
| PAT scope gate | MEDIUM | `POST /api/v1/tokens` is session-gated only. A `viewer` can request `atc:write` scope. Is this intentional (scope = capability grant regardless of role) or a gap? |
| Invite email delivery | MEDIUM | Invite `accept_url` is logged to console only. Resend key is configured (`RESEND_API_KEY`). Is there a sprint-planned integration? What is the current manual workaround? |
| Magic link replay guard | MEDIUM | `magic_link_tokens` records issuances but `consumed_at` is never stamped (callback does not write it). BR-040 claims replay guard exists — is this enforced only by Supabase Auth internally, and Bunkai's table is just an audit layer? |
| Workspace delete route | LOW | No `DELETE /api/v1/workspaces/{id}` route found. RLS policy `workspaces_delete_owner` exists in migration 0001. Is workspace deletion intentionally blocked at the API layer for MVP? |
| ATC `module_id` column | LOW | `AtcEditorPage` reads `atc.module_id` but the data map shows ATC belongs to `project_id` + `user_story_id`. Is `module_id` a denormalized convenience column added to the `atcs` table? The migration 0004 source was not read in detail. |
| Multi-workspace project URL shape | LOW | Current project route is `/projects/[projectSlug]` — no workspace context in URL. Comment in `ProjectPage` notes a future shape `/projects/{workspaceSlug}/{projectSlug}`. When does this migration happen? |
