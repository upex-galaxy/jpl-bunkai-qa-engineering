# Non-Functional Specs — Bunkai TMS

> Generated: 2026-06-19
> Source: `next.config.ts`, `supabase/migrations/`, `lib/env.ts`, `package.json` (target repo), App Router route structure
> Severity scale: CRITICAL · HIGH · MEDIUM · LOW

---

## NFR Summary

| ID | Category | Title | Current State | Severity |
|----|----------|-------|---------------|----------|
| NFR-001 | Performance | Response budget + ISR | GIN index present; no APM; no budget spec | MEDIUM |
| NFR-002 | Security | HTTP headers + rate limiting | Headers ABSENT; rate limiting ABSENT | HIGH |
| NFR-003 | Reliability | Error handling + uptime | No `error.tsx`; no RTO/RPO spec | HIGH |
| NFR-004 | Scalability | Pagination + connection limits | No pagination on ATC lists found | MEDIUM |
| NFR-005 | Observability | Logging + alerting | No structured logging; no APM; no alerting | HIGH |

---

## NFR-001 — Performance

### Current Implementation

| Aspect | State | Evidence |
|--------|-------|---------|
| ISR cache invalidation | Implemented — `revalidatePath` called on successful ATC save | `actions.ts:38` |
| ATC fulltext search index | Implemented — GIN index on `atcs.tsv` | `0004_atcs.sql` |
| DB connection pooling | Supabase managed (pgBouncer) | Supabase platform default |
| Image optimization | Next.js built-in (`<Image>`) — usage not confirmed in source | `next.config.ts` |
| Static rendering | Next.js 15 App Router — default behaviour; RSC for data-fetch routes | `package.json` next: 15.x |
| APM / RUM | **NOT found** | Discovery Gap |
| Performance budget | **NOT defined** | Discovery Gap |
| Bundle analysis | **NOT found** in scripts | `package.json` scripts |

### Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| No Web Vitals (LCP/FID/CLS) target | MEDIUM | Is there a CWV budget for the ATC editor page? |
| No APM (e.g., Vercel Analytics, Sentry Performance) | MEDIUM | Is Vercel Analytics enabled on the Vercel project dashboard? |
| ATC list pagination | MEDIUM | Are ATC lists paginated server-side or does the query return all rows? |
| DB query instrumentation | MEDIUM | Are slow queries surfaced anywhere? |

### QA Relevance

- Measure ATC editor page LCP at P75 as a regression baseline once staging URL is known.
- Verify GIN index is used (query plan) for fulltext searches exceeding 100 ATCs.
- Verify `revalidatePath` fires within 2s of a successful ATC save (ISR smoke test).

---

## NFR-002 — Security

### Current Implementation

| Control | State | Evidence |
|---------|-------|---------|
| Row-Level Security (RLS) | Implemented on ALL 12 tables | `supabase/migrations/0001–0012` |
| Supabase Auth | Magic link OTP (single-use replay guard) | `migrations/0009`; `middleware.ts` |
| PAT secret storage | SHA-256 hash only; prefix for O(1) lookup | `migrations/0008` |
| Invite token storage | SHA-256 hash only | `migrations/0010` |
| HTTPS | Enforced by Vercel (inferred) | `.env.example` Vercel hint |
| Cookie flags | `httpOnly` on `bk_active_ws`; Supabase SSR cookie | `middleware.ts`; `lib/supabase/` |
| HTTP security headers | **ABSENT** — `next.config.ts` has no `headers()` export | `next.config.ts` |
| Rate limiting | **ABSENT** — no middleware rate-limit; no Upstash/Arcjet import | `package.json`; `middleware.ts` |
| CSRF protection | Next.js Server Actions have built-in Origin header check (framework-level) | Next.js 15 docs |
| App-layer role check in Server Actions | **ABSENT** — `saveAtcAction` relies solely on RLS | `actions.ts:19–40` |
| Input sanitization | Zod validation at env and API route level; no XSS sanitization found for Markdown | `lib/env.ts` |
| Dependency audit | `bun audit` not in scripts | `package.json` |

### Missing HTTP Security Headers

The following headers are standard for Next.js production deployments and are **not configured**:

| Header | Recommended Value | Risk of Absence |
|--------|-------------------|-----------------|
| `Content-Security-Policy` | `default-src 'self'; script-src 'self' 'nonce-...'` | XSS |
| `X-Frame-Options` | `DENY` | Clickjacking |
| `X-Content-Type-Options` | `nosniff` | MIME sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Info leakage |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Feature creep |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Protocol downgrade |

> Fix: Add a `headers()` export to `next.config.ts` returning the above headers. CSP requires `nonce` support if inline scripts exist (shadcn/ui may inject them).

### Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| HTTP security headers absent | HIGH | Add `headers()` to `next.config.ts` |
| Rate limiting absent on `/api/auth/magic-link` | HIGH | Magic link endpoint has no throttle — brute-force email enumeration possible |
| Rate limiting absent on `/api/v1/` PAT routes | HIGH | API routes unthrottled |
| App-layer role check absent in `saveAtcAction` | MEDIUM | RLS is the only write guard — defence-in-depth recommends an explicit role check |
| Markdown / YAML content from ATC editor not sanitized | MEDIUM | XSS if rendered as HTML |
| `bun audit` not in CI/pre-commit | MEDIUM | Vulnerable deps may ship silently |

### QA Relevance

- **Critical security tests**: viewer role calling `saveAtcAction` → expect RLS 403.
- **Cross-workspace isolation**: workspace A member cannot read workspace B ATCs even via direct URL.
- **PAT scoping**: token with `atc:read` cannot write (403 on POST/PATCH ATC endpoints).
- **Token replay**: used magic link → second use → expect rejection.
- **Expired invite**: accept after `expires_at` → expect rejection.
- **Header check**: request to any page → `X-Frame-Options: DENY` present (blocked until headers added).

---

## NFR-003 — Reliability

### Current Implementation

| Aspect | State | Evidence |
|--------|-------|---------|
| Database transactions | Supabase-managed; RPC `bunkai_bootstrap_workspace` is SECURITY DEFINER (atomic) | `migrations/0006` |
| Server Action error return | `saveAtcAction` returns `{ ok: false, error }` on failure | `actions.ts` |
| `error.tsx` (App Router) | **NOT FOUND** — unhandled server errors surface as raw Next.js error page | App Router directory scan |
| `not-found.tsx` | **NOT CONFIRMED** — may exist but not verified in source | Discovery Gap |
| RTO target | **NOT DEFINED** | Discovery Gap |
| RPO target | **NOT DEFINED** | Discovery Gap |
| Supabase backup | Supabase platform default (daily backups on Pro plan) | Platform SLA |
| Retry logic | **NOT FOUND** — no retry on Server Actions or API routes | Source scan |

### Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| `error.tsx` missing | HIGH | Unhandled Server Component errors show raw stack trace in dev, blank screen in prod |
| RTO/RPO not defined | MEDIUM | What is the acceptable downtime / data loss window? |
| No retry on transient DB errors | MEDIUM | Supabase connection drops → user sees error with no retry |

### QA Relevance

- Test: force an invalid UUID in `saveAtcAction` → expect graceful `{ ok: false, error }` (not 500).
- Test: navigate to a non-existent ATC route → expect 404 page (verify `not-found.tsx` exists).
- Test: simulate Supabase auth session expiry → expect redirect to `/login?next=<path>`.

---

## NFR-004 — Scalability

### Current Implementation

| Aspect | State | Evidence |
|--------|-------|---------|
| Database | Supabase PostgreSQL (managed elastic scaling on Pro/Team plans) | Platform |
| Auth | Supabase Auth (stateless JWTs + refresh cookies) | `middleware.ts` |
| ATC fulltext index | GIN on `atcs.tsv` — scales logarithmically | `0004_atcs.sql` |
| API routes | Vercel Edge / Serverless (inferred) — stateless, auto-scale | Platform |
| ATC list pagination | **NOT OBSERVED** — no `LIMIT`/`OFFSET`/cursor seen on list routes | Source scan |
| File / attachment storage | **NOT FOUND** — no Supabase Storage usage found | Source scan |
| Workspace member limits | **NOT ENFORCED** — no `max_members` constraint found | Source scan |

### Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| No ATC list pagination | MEDIUM | Large projects with 1000+ ATCs will have slow list loads |
| Workspace size limits per plan | MEDIUM | Is `community` plan limited to N workspaces / N members? |
| Supabase connection pooling config | LOW | Using default pgBouncer? Custom pool size? |

### QA Relevance

- Load test: Workspace with 500+ ATCs — measure list endpoint P95 latency.
- Boundary: ATC with 100 steps → save → verify no truncation.
- Boundary: workspace slug at min length (3) and max length (40) — verify accepted.

---

## NFR-005 — Observability

### Current Implementation

| Aspect | State | Evidence |
|--------|-------|---------|
| Structured logging | **NOT FOUND** — no `pino`, `winston`, `consola` dependency | `package.json` |
| Error tracking | **NOT FOUND** — no Sentry / Bugsnag / Highlight import | `package.json` |
| APM / tracing | **NOT FOUND** — no OpenTelemetry, Datadog, Vercel Observability | `package.json` |
| Alerting | **NOT FOUND** — no PagerDuty / OpsGenie / Slack webhook | Source scan |
| Supabase dashboard metrics | Available (Supabase project dashboard) | Platform |
| Server Action `console.error` | Present in `actions.ts` (implicit — catch block standard) | Standard practice |

### Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| No error tracking service | HIGH | Unhandled errors in prod are invisible |
| No structured request logging | HIGH | No request ID, user ID, or latency in logs |
| No alerting on error rate | MEDIUM | How does on-call know when users hit 500s? |
| No distributed tracing | LOW | Cross-RPC trace linkage (middleware → Server Action → Supabase) not possible |

### QA Relevance

- Verify: after a failed `saveAtcAction`, the error is surfaced to the user (not silently swallowed).
- Verify: Supabase logs (dashboard) capture the failing RPC call on DB error.
- Monitor: error rate on `POST /api/v1/tokens` in Supabase dashboard during PAT auth tests.

---

## Compliance

| Area | State | Evidence |
|------|-------|---------|
| Data isolation | Implemented via RLS (workspace-scoped) | `supabase/migrations/0001–0012` |
| PII (email) | Stored in `auth.users` (Supabase managed) + `workspace_invites.email` | `migrations/0010` |
| GDPR right-to-erasure | **NOT IMPLEMENTED** — no `delete account` flow found | Source scan |
| SOC 2 | **NOT DOCUMENTED** — no compliance doc found | Source scan |
| HIPAA | Out of scope (no health data) | N/A |
| Audit log | **NOT FOUND** — no `audit_log` table or event stream | Source scan |

### Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| No account deletion / GDPR erasure flow | MEDIUM | Required for EU users under GDPR Art. 17 |
| No audit log | MEDIUM | Who changed what ATC and when? Required for enterprise QA workflows |
| `workspace_invites.email` stored plain | LOW | PII at rest — Supabase encrypts at storage layer but not field-level |

---

## Discovery Gaps — Master List

| Gap | Category | Severity | Source |
|-----|----------|----------|--------|
| HTTP security headers absent in `next.config.ts` | Security | HIGH | `next.config.ts` — no `headers()` export |
| Rate limiting absent on magic link + API routes | Security | HIGH | `middleware.ts` + `package.json` — no rate-limit dep |
| `error.tsx` missing from App Router | Reliability | HIGH | App Router directory — not found |
| No error tracking (Sentry / equivalent) | Observability | HIGH | `package.json` — no error-tracking dep |
| No structured logging | Observability | HIGH | `package.json` — no logging dep |
| ATC status transition mechanism missing | Functional | CRITICAL | No `runs` table; `AtcStatus` enum defined but untriggered |
| ATC list pagination absent | Scalability | MEDIUM | Route handlers — no `LIMIT`/cursor pattern observed |
| GDPR account deletion absent | Compliance | MEDIUM | No `/account/delete` or equivalent route |
| No audit log | Compliance | MEDIUM | No `audit_log` table in migrations |
| RTO/RPO not defined | Reliability | MEDIUM | No SLA doc found |
| `bun audit` not in scripts | Security | MEDIUM | `package.json` scripts |
| Plan feature limits not enforced | Business | MEDIUM | `WorkspacePlan` enum exists; no enforcement logic found |

---

## QA Relevance Summary

| NFR Area | Automated Tests | Manual / Exploratory |
|----------|----------------|---------------------|
| Security (RLS) | viewer write → 403; cross-workspace read → 0 rows | Manual role escalation attempts |
| Security (PAT) | `atc:read` token → POST ATC → 403; expired token → 401 | Token UI visibility (shown once) |
| Security (invite) | expired token accept → 422; reuse → 422 | Email delivery verification |
| Performance (ISR) | ATC save → cache invalidated within 2s | LCP on editor page (Lighthouse) |
| Performance (search) | 500 ATC search → P95 < 200ms | — |
| Reliability (error) | invalid UUID to saveAtcAction → graceful error | Direct DB error simulation |
| Reliability (session) | expired Supabase session → redirect to /login | Manual cookie expiry |
| Observability | — | Verify error appears in Supabase logs |
