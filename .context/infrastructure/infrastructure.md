# Infrastructure Mapping — Bunkai TMS

> Generated: 2026-06-19
> Source: `.env.example`, `package.json`, `supabase/migrations/`, `middleware.ts`, `next.config.ts`
> CI/CD: NONE (no `.github/workflows/` directory found)

---

## Environments

| Env | URL | Status | Notes |
|-----|-----|--------|-------|
| Local | `http://localhost:3000` | Confirmed | `bun run dev`; DB = local Supabase OR remote project |
| Staging | **UNKNOWN** | Discovery Gap | Vercel project URL needed; not in `.env.example` |
| Production | **UNKNOWN** | Discovery Gap | Vercel project URL needed; not in `.env.example` |

> For QA runs: default environment is **local** (`default_env: local` in `.agents/project.yaml`). Update to staging once URL is available.

---

## Deployment

| Aspect | Value | Evidence |
|--------|-------|---------|
| Platform | Vercel (inferred) | `.env.example` note: "Vercel + Supabase integration"; `outputFileTracingRoot` in `next.config.ts` |
| Deploy trigger | Push to main (inferred) | No workflow file — likely Vercel GitHub integration |
| Build command | `bun run build` (`next build`) | `package.json` scripts |
| Environment management | Vercel dashboard + `.env` locally | `.env.example` |
| Preview deployments | Vercel PR previews (likely enabled — standard Vercel) | Inferred |
| Rollback | Vercel deployment history (instant rollback via dashboard) | Vercel platform |

---

## CI/CD

| Aspect | State |
|--------|-------|
| CI system | **NONE** — no `.github/workflows/` directory |
| Pre-commit hooks | Husky + lint-staged (local only) |
| Pre-push hooks | Unknown — not confirmed |
| Automated tests in CI | N/A (no test runner) |
| Linting in CI | **NOT CONFIGURED** |
| Type-checking in CI | **NOT CONFIGURED** |
| Build verification | Vercel build pipeline (not GitHub Actions) |

> **Gap (HIGH)**: No CI pipeline means broken code can merge. Recommend: GitHub Actions workflow running `bun run repo:check` + `bun run build` on PR. Add Playwright E2E job once `/adapt-framework` completes.

---

## Backend Services

### Supabase (Database + Auth)

| Aspect | Value | Evidence |
|--------|-------|---------|
| Product | Supabase Cloud (hosted) | `NEXT_PUBLIC_SUPABASE_URL` env var |
| Database | PostgreSQL (managed) | All migrations in `supabase/migrations/` |
| Auth | Supabase Auth (Magic Link OTP) | `@supabase/ssr`, `middleware.ts` |
| RLS | Enforced on all 12 tables | Migrations 0001–0012 |
| Connection method | `@supabase/ssr` SSR client (cookie-based session) | `middleware.ts` |
| Programmatic access | `@supabase/supabase-js` (service role for admin ops) | `lib/env.ts` SUPABASE_SERVICE_ROLE_KEY |
| Local development | Supabase CLI local stack (inferred — no `config.toml` found in glob) | Discovery Gap |
| Migrations | 12 SQL files applied sequentially | `supabase/migrations/` |

### Resend (Transactional Email)

| Aspect | Value | Evidence |
|--------|-------|---------|
| Product | Resend | `.env.example` RESEND_API_KEY |
| Use cases | Magic link delivery, workspace invite emails | Auth flow + invite flow |
| SDK | Unknown — direct Resend API or @resend/node (not found in package.json deps) | Discovery Gap |

### OpenAPI / Scalar

| Aspect | Value |
|--------|-------|
| Spec generation | `bun run openapi:gen` → `bun scripts/openapi-gen.ts` (Zod → OpenAPI via `@asteasolutions/zod-to-openapi`) |
| Spec diff | `bun run openapi:diff` (detects breaking changes between versions) |
| Docs UI | Scalar (`@scalar/api-reference-react`) at `/api/docs` |
| QA type sync | `bun run api:sync` → `bun scripts/sync-openapi.ts` → `api/openapi-types.ts` in QA repo |

---

## Infrastructure Topology

```
+--------------------+       +---------------------+       +------------------+
|   Browser / User   |       |    Vercel Edge       |       |  Supabase Cloud  |
|                    |       |  (Next.js App Router)|       |                  |
|  /projects/...     +------>+  middleware.ts       +------>+  PostgreSQL      |
|  /api/v1/...       |       |  Server Components   |       |  RLS enforced    |
|  /api/docs         |       |  Server Actions      |       |  12 tables       |
+--------------------+       |  Route Handlers      |       |  8 RPCs          |
                             +----------+-----------+       +------------------+
                                        |
                             +----------v-----------+       +------------------+
                             |   External Services  |       |   Resend         |
                             |                      +------>+  Transactional   |
                             |   PAT → Bearer auth  |       |  Email           |
                             |   Magic Link OTP     |       +------------------+
                             +----------------------+
```

---

## Monitoring & Observability

| Aspect | State | Evidence |
|--------|-------|---------|
| Error tracking | **NONE** | No Sentry/Bugsnag dep in `package.json` |
| APM | **NONE** | No OpenTelemetry/Datadog dep |
| Structured logging | **NONE** | No pino/winston/consola dep |
| Uptime monitoring | **UNKNOWN** | Discovery Gap |
| Alerting | **UNKNOWN** | Discovery Gap |
| Supabase dashboard | Available (DB metrics, auth logs, RLS audit) | Supabase Cloud platform |
| Vercel dashboard | Available (function logs, build logs, analytics) | Vercel platform |

---

## Security Posture

| Control | State | Evidence |
|---------|-------|---------|
| HTTPS | Enforced by Vercel | Platform |
| RLS | All 12 tables | Migrations |
| HTTP security headers | **ABSENT** | `next.config.ts` — no `headers()` export |
| Rate limiting | **ABSENT** | No middleware rate-limit |
| Session storage | httpOnly cookie (Supabase SSR) | `middleware.ts` |
| Secrets in repo | None found | `.gitignore` covers `.env` |
| Dependency auditing | **NOT IN SCRIPTS** | `package.json` — no `bun audit` step |

---

## Local Development Setup

```bash
# 1. Clone target repo
git clone <repo-url> c:/Projects/UPEX/upex-bunkai-tms
cd c:/Projects/UPEX/upex-bunkai-tms

# 2. Install dependencies
bun install

# 3. Configure environment
cp .env.example .env
# Edit .env:
#   NEXT_PUBLIC_SUPABASE_URL=<your-supabase-url>
#   NEXT_PUBLIC_SUPABASE_ANON_KEY=<your-anon-key>   ← use this key name (not SUPABASE_PUBLISHABLE_KEY)
#   SUPABASE_SERVICE_ROLE_KEY=<service-role-key>
#   RESEND_API_KEY=<resend-key>

# 4. Apply DB migrations
#    Option A: Supabase local (if config.toml exists)
#    bunx supabase start
#    bunx supabase db push
#    Option B: Remote Supabase project
#    bunx supabase db push --db-url <connection-string>

# 5. Generate Supabase TypeScript types
bun run types:gen

# 6. Sync OpenAPI types to QA repo (run from QA repo)
bun run api:sync

# 7. Start dev server
bun run dev
# → http://localhost:3000
```

---

## Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| Staging URL unknown | HIGH | What is the Vercel staging deployment URL? |
| Production URL unknown | HIGH | What is the production URL? |
| No CI/CD pipeline | HIGH | Will GitHub Actions be added? Target: `bun run repo:check` + `bun run build` on PR |
| Supabase local dev (no `config.toml` found) | MEDIUM | Is `supabase start` used locally or does the team connect to a remote Supabase project? |
| Resend SDK not in `package.json` deps | MEDIUM | Is Resend called via raw `fetch` or a separate package? |
| `SUPABASE_PUBLISHABLE_KEY` in `.env.example` | MEDIUM | Should `.env.example` be updated to `NEXT_PUBLIC_SUPABASE_ANON_KEY`? |
| No rollback procedure documented | MEDIUM | How is a bad deploy rolled back? Vercel dashboard? Any automated gates? |
