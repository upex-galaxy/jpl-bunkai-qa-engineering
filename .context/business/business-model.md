# Business Model — Bunkai (分解)

> Confidence: **High** for core product, **Low** for revenue model (no payment code found)
> Generated: 2026-06-19
> Source: Reverse-engineered from `c:/Projects/UPEX/upex-bunkai-tms/` (DESIGN.md, lib/types.ts, supabase/migrations/, app/ routes)

---

## Problem Statement

QA teams that practice structured acceptance testing face a fragmentation problem: acceptance criteria live in Jira, test cases live in spreadsheets or ad-hoc documents, and run results live in screenshots or informal reports. There is no single place that anchors a **test case to the specific acceptance criterion it verifies**, making traceability between requirements and test evidence expensive to maintain.
(Source: DESIGN.md §1 "developer-first, opinionated about quality"; `lib/types.ts` — `AtcAcceptanceCriterion` M:N join table; migration 0004 comment "anchoring moat")

Bunkai (分解 — the martial-arts practice of breaking down a kata into its component moves) is a purpose-built Test Management System that makes the ATC↔AC link a first-class, enforced data constraint — not a convention.
(Source: DESIGN.md §1 brand definition; migration 0004 `atc_acceptance_criteria` table comment)

The product targets engineering teams that already use Jira for requirements and want to keep test evidence in a dedicated, version-controlled store rather than Jira custom fields.
(Source: `.agents/project.yaml` `issue_tracker: Jira`; `app/(app)/projects/[projectSlug]/atcs/[atcId]/actions.ts`)

---

## Business Model Canvas

### 1. Customer Segments

| Segment | Evidence | Confidence |
|---------|----------|------------|
| QA engineers / SDETs | `MemberRole` includes `member` (primary QA role); ATC authoring UI is the core feature | High |
| QA leads / managers | `admin` and `owner` roles with member management; workspace invites | High |
| Read-only stakeholders (PMs, devs) | `viewer` role in RBAC | High |
| Teams using Jira for requirements | `.agents/project.yaml` Jira integration; external_id + external_url on UserStory | High |
| Found in: `lib/types.ts` MemberRole enum; `supabase/migrations/0001_tenancy.sql` |  |  |

### 2. Value Propositions

| Proposition | Evidence | Confidence |
|-------------|----------|------------|
| ATC anchored to AC — traceability enforced, not optional | `atc_acceptance_criteria` M:N join (migration 0004) | High |
| Multi-layer test cases (UI/API/Unit in one tool) | `AtcLayer` enum `'UI' \| 'API' \| 'Unit'`; layer chips in DESIGN.md | High |
| Structured step-by-step test cases with input data + expected output | `atc_steps` table (content, input_data, expected) | High |
| Fulltext ATC search with tag-prefix queries | `atcs.tsv` GIN index (migration 0004) | High |
| Multi-tenant workspaces with RBAC | `workspaces` + `workspace_members` with RLS (migration 0001) | High |
| Open REST API with live OpenAPI docs | `/api/v1/` routes + `/api/docs` Scalar UI | High |
| Found in: migrations 0001–0004; DESIGN.md; lib/types.ts |  |  |

### 3. Channels

| Channel | Evidence | Confidence |
|---------|----------|------------|
| Web app (SaaS) | Next.js 15 app, Vercel deploy | High |
| REST API (headless / automation integration) | `/api/v1/` + PAT access tokens (migration 0008) | High |
| Email invites | `workspace_invites` (migration 0010); magic link auth | High |
| Found in: app/ routes; `.env.example` RESEND_API_KEY; supabase/migrations/0008, 0010 |  |  |

### 4. Customer Relationships

| Type | Evidence | Confidence |
|------|----------|------------|
| Self-service | Magic link login, onboarding form, workspace auto-creation | High |
| Invite-based team onboarding | `workspace_invites` + accept flow (`/invites/accept`) | High |
| Found in: `app/(auth)/login/`, `app/(app)/onboarding/`, `app/invites/accept/` |  |  |

### 5. Revenue Streams

| Stream | Evidence | Confidence |
|--------|----------|------------|
| SaaS subscription (community / cloud / enterprise tiers) | `WorkspacePlan` enum; plan stored on workspace | Medium |
| Per-plan feature limits | **Unknown** — plan check not found in source (enforcement likely future) | Low |
| Found in: `lib/types.ts` WorkspacePlan; supabase/migrations/0001 plan column |  |  |
| **Unknown**: No Stripe/Paddle/LemonSqueezy deps found. Billing not yet implemented. |  |  |

### 6. Key Resources

| Resource | Evidence | Confidence |
|----------|----------|------------|
| Supabase PostgreSQL (data + auth) | All migrations + `lib/supabase/` | High |
| Next.js / Vercel runtime | `next.config.ts`, `.env.example` Vercel hints | High |
| OpenAPI spec + generated types | `/api/openapi` route, `bun run api:sync` | High |
| Found in: package.json; `.env.example`; `lib/env.ts` |  |  |

### 7. Key Activities

| Activity | Evidence | Confidence |
|----------|----------|------------|
| ATC authoring (create, edit, save steps + assertions) | `app/(app)/projects/[projectSlug]/atcs/[atcId]/actions.ts`; migration 0007 | High |
| ATC execution tracking (status transitions) | `AtcStatus` enum; migration 0004 | High |
| Workspace + project management | `workspaces`, `projects`, `modules` tables | High |
| User story + AC management | `user_stories`, `acceptance_criteria` tables | High |
| Workspace member invites | migration 0010 | High |
| API token management | migration 0008, 0011, 0012 | High |
| Found in: supabase/migrations/; app/ routes |  |  |

### 8. Key Partners

| Partner | Evidence | Confidence |
|---------|----------|------------|
| Supabase (DB + Auth + MCP) | Primary backend — all data + auth flows | High |
| Vercel (hosting + edge) | `.env.example` Vercel+Supabase integration note | Medium |
| Resend (transactional email) | `.env.example` RESEND_API_KEY; magic link delivery | High |
| Atlassian / Jira (requirements source) | `.agents/project.yaml`; UserStory.external_id | High |
| Found in: `.env.example`; package.json; .agents/project.yaml |  |  |

### 9. Cost Structure

| Cost | Evidence | Confidence |
|------|----------|------------|
| Supabase hosting (DB, Auth, storage) | Primary infra cost | High |
| Vercel hosting (compute, edge, CDN) | Inferred from deploy pattern | Medium |
| Resend email sending | Transactional emails per invite/magic-link | Medium |
| Found in: `.env.example`; DESIGN.md §source files |  |  |

---

## Discovery Gaps

- [ ] **Revenue model**: `WorkspacePlan` enum exists but no billing/payment code found. Monetization may be planned (not yet implemented).
- [ ] **Plan feature limits**: What features are gated per plan (community / cloud / enterprise)?
- [ ] **Staging/production URLs**: Vercel project URL not in source.
- [ ] **Market positioning**: No competitor comparison found. DESIGN.md mentions "inspired by Linear, VS Code, GitHub, Vercel Dashboard" in design terms only.

---

## QA Relevance

| Business Aspect | Testing Implication |
|-----------------|---------------------|
| ATC↔AC anchoring moat | Test that ATCs cannot be saved without ≥1 AC link |
| Multi-tenant RLS | Test data isolation: workspace A cannot read workspace B data |
| Role-based mutations | Test `viewer` cannot create/edit ATCs; `member` can; `owner` can manage members |
| ATC status transitions | Test all 6 status values and illegal transitions |
| Invite flow | Test invite email delivery, accept link expiry, duplicate acceptance |
| PAT API auth | Test that expired/revoked tokens return 401; valid tokens return data |
| Magic link auth | Test link expiry, reuse, redirect-after-login |
| OpenAPI spec accuracy | Contract testing: spec at `/api/openapi` matches actual route behavior |

---

## Sources Used

1. `DESIGN.md` — brand, product name, design principles, entity IDs (ATC-XXX, MOD-XXX)
2. `lib/types.ts` — canonical TypeScript interfaces for all entities
3. `supabase/migrations/0001_tenancy.sql` — workspaces + workspace_members schema + RLS
4. `supabase/migrations/0004_atcs.sql` — atcs + atc_steps + atc_assertions + atc_acceptance_criteria
5. `app/` directory listing — route map (onboarding, projects, atcs, workspaces, auth, invites, qa, api)
6. `.env.example` — infrastructure partners (Supabase, Vercel, Resend, n8n)
7. `.agents/project.yaml` — Jira integration, project key BK
