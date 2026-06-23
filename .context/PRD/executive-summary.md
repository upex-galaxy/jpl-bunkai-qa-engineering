# Executive Summary — Bunkai TMS

> Phase 2 PRD document. Generated from source-code analysis of `upex-bunkai-tms`.
> Sibling docs: `user-personas.md`, `user-journeys.md`.
> Feature catalog: `.context/business/business-feature-map.md` (separate command).

---

## 1. Problem Statement

### The Challenge

QA engineering teams struggle to maintain a stable, traceable link between what a product is *supposed* to do (Acceptance Criteria in user stories) and what is actually *tested* (test cases and their execution results). In traditional test management tools, test cases are authored in isolation — engineers copy-paste criteria from Jira tickets into a spreadsheet or TMS form with no enforced relationship. Over time, the test suite drifts: ACs are updated, bugs are fixed, and nobody knows which test cases are affected.

The second challenge is execution fragmentation. Manual testers run the same suite in one tool, automation engineers have Playwright specs in a repo, and CI pipelines produce separate JUnit XML reports. There is no single source of truth for the current quality state of a feature — only tribal knowledge and periodic sync-up meetings.

The third challenge is the "black box" nature of existing coverage metrics. Tools that report "80% of test cases passed" do not answer the deeper question: *which Acceptance Criteria are actually covered?* Coverage that is not anchored to product requirements is meaningless for product owners and engineering managers who need to make go/no-go decisions.

Source: product copy on `/login` page (`app/(auth)/login/page.tsx`), onboarding form copy (`app/(app)/onboarding/onboarding-form.tsx`), and DESIGN.md §2 design principles.

### Current Alternatives

No direct competitive analysis is present in the codebase. The login page copy acknowledges the self-hosted option explicitly (`Terminal` card: "Connect to your own Bunkai server — Community edition"), suggesting the product positions itself against hosted-only TMS tools (TestRail, Zephyr, qTest). The login page mentions Apache-2.0 and `docker compose up` as differentiators, implying the alternatives require paid cloud subscriptions and offer no self-hosting.

Source: `app/(auth)/login/page.tsx` lines 98–106, 175–179.

---

## 2. Solution Overview

### Product Vision

Bunkai (分解) is an open-source, self-hostable Test Management System that enforces a traceable link from every Acceptance Test Case to at least one Acceptance Criterion — making coverage provable, not assumed.

### Core Capabilities

| # | Feature | Problem Addressed | Evidence |
|---|---|---|---|
| 1 | **AC Anchoring (Moat)** | Test cases drift from requirements | `AnchoringPanel.tsx`; `actions.ts` validation: `acIds.length === 0` blocks save |
| 2 | **IQL Lifecycle** | No shared model across manual, agentic, CI execution | Login page FEATURE_TICKS: "Manual · Agentic · CI execution. Same schema, same reports." |
| 3 | **ATC Editor with Monaco** | Authoring experience limited to forms/JIRA fields | `AtcEditor.tsx` — StepEditor (Monaco), assertions YAML, tags, layer selector |
| 4 | **Multi-tenant Workspaces + RBAC** | Team-level isolation lacking in open tools | `lib/types.ts` `WorkspaceMember`, `MemberRole`; `workspaces/route.ts` RLS |
| 5 | **PAT-based API Access** | Automation pipelines blocked behind browser sessions | `tokens/route.ts` — `bk_pat_` family, scopes: `atc:read`, `atc:write`, `run:execute`, `workspace:admin` |

### Key Differentiators

- **Anchoring moat**: an ATC cannot be saved without binding ≥1 Acceptance Criterion. This is not a guideline — it is a schema-level enforcement that blocks the save action. Source: `actions.ts` lines 29–31; `AtcEditor.tsx` `canSave` logic.
- **Execution convergence**: the IQL methodology targets manual, agentic, and CI execution against the same schema, eliminating three separate result stores. Source: login page FEATURE_TICKS `×3`.
- **Self-hosted by default**: `docker compose up` and Apache-2.0 license. Cloud is an opt-in tier, not the default. Source: login page footer, `app/(auth)/login/page.tsx` lines 98–106.
- **Developer-first density**: design principles reject whitespace-heavy consumer-app aesthetics in favor of VS Code / Linear information density for QAs managing hundreds of ATCs daily. Source: `DESIGN.md` §2.

---

## 3. Success Metrics

### Tracked Metrics (Analytics Instrumentation)

**None detected.** A search for `track(`, `analytics.`, `posthog`, `mixpanel`, `segment`, `gtag` across `app/` and `lib/` returned no results. No analytics SDK is wired up in the current codebase.

### Inferred KPIs (from feature set)

| KPI | Derivation |
|---|---|
| ATC Anchoring Rate | % of ATCs with ≥1 bound AC — enforced by schema, measurable from `atc_acceptance_criteria` table |
| Mean ATCs per User Story | Depth of coverage per story — queryable from `atcs` + `user_stories` tables |
| ATC Status Distribution | pass / fail / blocked / skipped / unrun breakdown — `Atc.status` field |
| Time-to-First-ATC | Minutes from workspace creation to first ATC save — derivable from `workspaces.created_at` + `atcs.created_at` |
| Team Collaboration Rate | % of workspaces with >1 active member — from `workspace_members` |
| Invite Conversion Rate | Invites issued vs. accepted — `workspace_invites.accepted_at` vs. total |
| PAT Adoption | % of active members with ≥1 PAT — from `access_tokens` |

### Unknown Metrics (Gaps)

- No run/execution entity exists yet in the schema (`Atc.status` is set directly; no `Run` or `TestExecution` table found). The DESIGN.md §12 references a "run" screen in mockups but no `Run` entity is present in `lib/types.ts`. Execution lifecycle metrics are currently untrackable.
- No session replay, error tracking (Sentry), or uptime monitoring instrumentation found.

---

## 4. Target Users

Detailed personas are in `user-personas.md`. Brief summary:

| Persona | System Role | Primary Need |
|---|---|---|
| Workspace Owner | `owner` | Create and own the team's QA workspace; invite teammates |
| QA Lead / Admin | `admin` | Invite members, manage workspace membership, oversee test coverage |
| QA Engineer | `member` | Author ATCs, link them to ACs, run and record test results |
| Stakeholder / Observer | `viewer` | Read-only visibility into test coverage and results |

Source: `lib/types.ts` `MemberRole`; `workspaces/[id]/invites/route.ts` role enum.

---

## 5. Product Scope

### What's Included (Current — v0.1.0)

- Magic-link authentication via Supabase Auth (email/password path also in API: `auth/signup/route.ts`, `auth/signin/route.ts`)
- Workspace creation and multi-workspace switching
- Project management within workspaces
- Module tree (hierarchical folder structure for ATCs)
- User Story + Acceptance Criterion management
- ATC authoring: title, layer (UI/API/Unit), steps (Markdown), assertions (YAML), tags, AC anchoring
- ATC status tracking: unrun / running / pass / fail / blocked / skipped
- Team invites with token-based acceptance and RBAC roles
- Personal Access Tokens (PATs) with scoped permissions
- REST API v1 (documented at `/api/docs` via Scalar, OpenAPI JSON at `/api/openapi`)
- Command palette (`Cmd/Ctrl+K`)
- In-app testability guide (`/qa` route)
- Design token preview page (`/design-tokens`)
- Self-hosting via `docker compose`

### What's Not Included (Known Limits)

- **OAuth / SSO**: GitHub and Google buttons present on login page but explicitly disabled ("soon" label). Source: `app/(auth)/login/page.tsx` lines 134–155.
- **Email sending for invites**: `POST /api/v1/workspaces/{id}/invites` logs the accept URL to server console only. "MVP does not send an email." Source: `workspaces/[id]/invites/route.ts` line 79.
- **Test Execution / Run entity**: No `Run`, `TestRun`, or `TestExecution` table in `lib/types.ts`. ATC status is set directly on the ATC, not on a separate execution record. Execution history is not tracked.
- **Multi-workspace route shape**: current routes use `/projects/{projectSlug}` without a workspace prefix. A comment in `projects/[projectSlug]/page.tsx` line 23–25 notes the future route will be `/projects/{workspaceSlug}/{projectSlug}`.
- **Light mode**: `DESIGN.md` §9 states "Light mode follows in Phase 2."
- **Mind-map view**: referenced in `DESIGN.md` §6 ("view-toggle: Tree / Table / Mind-map") but not implemented.
- **Bulk operations**: no bulk ATC create, bulk status update, or import.
- **Reporting / Dashboards**: no aggregate coverage or run history views found.

### Future Indicators

| Indicator | Evidence |
|---|---|
| `Run` / `TestExecution` entity | DESIGN.md §11 references a "run" screen in mockups; `DESIGN.md` §6 mentions `.bar.seg-bar` multi-segment progress bars for status breakdowns — not yet wired |
| Multi-workspace route prefix | Comment in `projects/[projectSlug]/page.tsx` lines 23–25 |
| OAuth (GitHub, Google) | Login page disabled buttons with "soon" label |
| Transactional email | `invites/route.ts` comment: "MVP does not send an email" |
| Mind-map view | `DESIGN.md` §6 segmented control "Tree / Table / Mind-map" |
| Light theme | `DESIGN.md` §9 "Light theme (Phase 2)" |
| ATC versions / history | `Atc.version: number` field in `lib/types.ts` — field exists but no version history UI found |

---

## 6. Discovery Gaps

| Gap | Impact | Suggested Source |
|---|---|---|
| No analytics instrumentation | Cannot measure activation, engagement, or retention without future instrumentation | Ask product owner for analytics strategy |
| No `Run`/`TestExecution` entity in schema | Execution lifecycle, pass rate trends, and per-run history are unspecifiable | Confirm with team: is this Phase 2 or deferred? |
| Permission matrix not explicit in code | RBAC behavior is partially enforced via Supabase RLS (not visible in app code) | Read DB migration files or Supabase RLS policies |
| Project creation flow not found | How does a new `Project` get created? No UI or API route found for `POST /projects` | Search for project creation form or API endpoint |
| Module creation flow not found | How do `Module` records get created? No module creation UI found | Search for module creation API or form |
| UserStory + AC creation flow | No UI for creating user stories or ACs was found in the app routes | Confirm whether this is done via external sync (Jira) or planned in-app UI |
| `workspace:admin` PAT scope behavior | What actions does this scope unlock beyond `atc:read/write` and `run:execute`? | Read `lib/api/auth.ts` scope resolution |
| Invite role cap | Can an `admin` invite another `admin`? The schema allows `viewer/member/admin` only (owner excluded). Who can issue invites? | Read RLS policy on `workspace_invites` |

---

## 7. QA Relevance

### Critical Testing Areas

| Area | Why Critical |
|---|---|
| AC Anchoring enforcement | Core product moat — save must fail without bound AC; test both UI block and API bypass attempt |
| Invite + acceptance flow | Token-based, one-time-visible URL, email-match gating, expiry, revoke — 5 distinct failure modes |
| PAT issuance and auth | `bk_pat_` secret shown once; hash stored separately; scopes gate API endpoints |
| RBAC boundaries | RLS in Supabase gates mutations; test that `viewer` cannot write and `member` cannot invite |
| Workspace isolation (multi-tenancy) | RLS must prevent cross-workspace data leaks; critical security boundary |
| ATC save validation | Three required fields (title, user story, ≥1 AC) each independently block save |

### Risk Areas

| Risk | Severity |
|---|---|
| PAT secret leaked post-issuance (single-display window) | High — no retrieval path; user must regenerate |
| Invite token not email-gated if auth check bypassed | High — any authenticated user could accept any invite |
| Cross-workspace project lookup | High — `projects/[projectSlug]/page.tsx` relies on RLS; slug uniqueness across workspaces unverified |
| `owner` role cannot be invited (invite schema caps at `admin`) | Medium — escalation path for workspace ownership transfer unclear |
| ATC version field incremented but no version history UI | Low now, high when version rollback becomes expected |

---

## 8. Document References

| Document | Path | Status |
|---|---|---|
| User Personas | `.context/PRD/user-personas.md` | Generated (Phase 2) |
| User Journeys | `.context/PRD/user-journeys.md` | Generated (Phase 2) |
| Business Feature Map | `.context/business/business-feature-map.md` | Pending (`/business-feature-map` command) |
| Business Data Map | `.context/business/business-data-map.md` | Pending (`/business-data-map` command) |
| Business API Map | `.context/business/business-api-map.md` | Pending (`/business-api-map` command) |
| Master Test Plan | `.context/master-test-plan.md` | Exists |
| Architecture Specs | `.context/SRS/architecture-specs.md` | Exists |
