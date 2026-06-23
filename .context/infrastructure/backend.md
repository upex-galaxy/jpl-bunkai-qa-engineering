# Backend Infrastructure — Bunkai TMS

> Generated: 2026-06-19
> Source: `package.json`, `next.config.ts`, `middleware.ts`, `supabase/migrations/`, `lib/env.ts`

---

## Runtime

| Aspect | Value | Evidence |
|--------|-------|---------|
| Language | TypeScript 5.9 (`strict: true`) | `package.json` devDeps; `tsconfig.json` strict flag |
| Runtime | Bun | `@types/bun` in devDeps; scripts use `bun scripts/...` |
| Framework | Next.js 15 (App Router) | `package.json` `next: "^15"` |
| Rendering model | React Server Components (RSC) first; Server Actions for mutations | `app/(app)/*/actions.ts` pattern |
| Node compat | Node.js compatible (Next.js runtime) — Bun is the CLI/script runner | `next.config.ts` `outputFileTracingRoot` |

---

## Build Configuration

```bash
# Install dependencies
bun install

# Development server (hot-reload, localhost:3000)
bun run dev

# Production build
bun run build

# Production server (after build)
bun run start

# Type-check only (no emit)
bun run typecheck        # alias: bun run types:check

# Full repo health check (format + lint + types + vars + skills)
bun run repo:check

# Fix all auto-fixable issues
bun run repo:fix
```

---

## Database

| Aspect | Value | Evidence |
|--------|-------|---------|
| Provider | Supabase (hosted PostgreSQL) | `@supabase/ssr`, `@supabase/supabase-js` |
| ORM | None — direct Supabase client + RPC calls | `lib/supabase/` client, `saveAtc()` RPC pattern |
| Connection | `@supabase/ssr` — SSR-aware cookie session client | `middleware.ts` |
| Pooling | pgBouncer (Supabase managed) | Platform default |
| Migrations | SQL files — `supabase/migrations/0001–0012.sql` | Migration directory |
| Schema management | Applied via Supabase CLI (`supabase db push` or local migration apply) | Convention |
| Type generation | `bun run types:gen` → `bun scripts/gen-supabase-types.ts` | `package.json` scripts |

### Migration Summary

| File | Purpose |
|------|---------|
| `0001_tenancy.sql` | workspaces, workspace_members, RLS policies |
| `0002_projects_modules.sql` | projects, modules tables |
| `0003_authoring.sql` | user_stories, acceptance_criteria tables |
| `0004_atcs.sql` | atcs, atc_steps, atc_assertions, atc_acceptance_criteria; GIN index on tsv |
| `0005_rls_helpers.sql` | Helper functions: `bunkai_is_workspace_member`, `bunkai_is_workspace_admin` |
| `0006_bootstrap_workspace.sql` | `bunkai_bootstrap_workspace` SECURITY DEFINER RPC |
| `0007_save_atc.sql` | `bunkai_save_atc` (SECURITY INVOKER) — atomic ATC upsert |
| `0008_access_tokens.sql` | access_tokens table + scopes CHECK constraints |
| `0009_cross_cutting.sql` | magic_link_token_secrets (OTP replay guard) |
| `0010_workspace_invites.sql` | workspace_invites table + invite lifecycle |
| `0011_split_token_secrets.sql` | token_prefix / hash split for PAT storage |
| `0012_drop_legacy_token_hashes.sql` | Removes deprecated token storage columns |

---

## Authentication

| Flow | Implementation |
|------|----------------|
| Primary auth | Magic Link (Supabase OTP via email — Resend or Supabase SMTP) |
| Session storage | `@supabase/ssr` httpOnly cookie (set/refreshed in `middleware.ts`) |
| Session refresh | `supabase.auth.getUser()` called on every request in middleware |
| Route protection | `middleware.ts` — checks `PROTECTED_PREFIXES = ['/projects', '/onboarding']` |
| Unauthenticated redirect | `→ /login?next=<original-path>` |
| Public routes | `/login`, `/auth/*`, `/api/auth/*` (no auth check) |
| API auth (programmatic) | PAT Bearer token `bk_pat_<prefix>.<secret>` (SHA-256 hash, O(1) lookup) |
| Auth middleware matcher | All routes except `_next/static`, `_next/image`, `favicon.ico`, static files |

```
Request → middleware.ts
  ├── Create Supabase SSR client
  ├── supabase.auth.getUser()      ← refreshes session if cookie expired
  ├── isProtected(pathname)?
  │     └── !user → redirect /login?next=<path>
  └── NextResponse.next()          ← pass through with updated cookies
```

---

## API Layer

| Aspect | Value |
|--------|-------|
| API location | `app/api/v1/` — Next.js Route Handlers |
| API auth | PAT Bearer token (programmatic) OR session cookie (browser) |
| OpenAPI spec | Generated via `bun run openapi:gen` (`@asteasolutions/zod-to-openapi`) |
| API docs UI | `@scalar/api-reference-react` at `/api/docs` |
| Type sync to QA repo | `bun run api:sync` → `bun scripts/sync-openapi.ts` |
| Typed routes | `typedRoutes: true` in `next.config.ts` — all `href` values are type-checked |

---

## Environment Variables

Validated at startup via `lib/env.ts` (Zod schema, server-only).

| Variable | Description | Used in |
|----------|-------------|---------|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL | `middleware.ts`, Supabase clients |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon key (public) | `middleware.ts` |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role (server only) | Admin-level DB operations |
| `RESEND_API_KEY` | Resend transactional email | Magic link / invite email delivery |

> **Inconsistency (Discovery Gap)**: `.env.example` uses `SUPABASE_PUBLISHABLE_KEY`; `lib/env.ts` and `middleware.ts` use `NEXT_PUBLIC_SUPABASE_ANON_KEY`. Canonical name is `NEXT_PUBLIC_SUPABASE_ANON_KEY` (confirmed in source code). `.env.example` key name is stale.

For QA boilerplate `.env` and `.agents/project.yaml` env resolution, use:
- `LOCAL_USER_EMAIL` / `LOCAL_USER_PASSWORD` for local test accounts
- `STAGING_USER_EMAIL` / `STAGING_USER_PASSWORD` for staging

---

## Code Quality Pipeline

```bash
# Run in order (also what bun run repo:check does):
bun run format:check   # Prettier — JSON, YAML, CSS, HTML
bun run lint:check     # ESLint (@antfu/eslint-config + @next/eslint-plugin-next)
bun run types:check    # tsc --noEmit
bun run vars:check     # lint-vars (project variable references)
bun run vars:env:check # check .env completeness
bun run skills:check   # lint skill files
bun run skills:registry:check  # verify skill registry is up-to-date
```

Pre-commit hooks (Husky + lint-staged):
- `*.{ts,tsx,js,jsx}` → `eslint --fix`
- `*.{json,yml,yaml,css,scss,html}` → `prettier --write`

---

## Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| `lib/` directory empty in glob (env var inconsistency confirmed) | MEDIUM | Confirm `NEXT_PUBLIC_SUPABASE_ANON_KEY` is the canonical name in `.env` |
| No test runner configured | CRITICAL | No Jest, Vitest, or Playwright in dependencies — requires `/adapt-framework` |
| Supabase CLI version / local dev config | MEDIUM | Is `supabase start` used for local development? Is there a `supabase/config.toml`? |
| `bunkai_save_atc` RPC internals (0007) | HIGH | Steps + assertions + AC links — are they upserted atomically? What are rollback cases? |
| ATC creation route | HIGH | How is a new `atcs` row first created? No "new ATC" route found. |
