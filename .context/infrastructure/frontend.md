# Frontend Infrastructure — Bunkai TMS

> Generated: 2026-06-19
> Source: `package.json`, `tailwind.config.ts`, `next.config.ts`, `middleware.ts`, `DESIGN.md`

---

## Runtime

| Aspect | Value | Evidence |
|--------|-------|---------|
| Framework | Next.js 15 (App Router) | `package.json` `next: "^15"` |
| React version | React 19 | `package.json` `react: "^19"` |
| Rendering model | Server Components (RSC) first; `'use client'` only for interactive leaves | App Router convention |
| Routing | File-based; typed routes (`typedRoutes: true`) | `next.config.ts` |
| Bundler | Next.js built-in (Turbopack in dev via `next dev`) | `next.config.ts` |
| State management | None (no Redux, Zustand, Jotai found) — RSC + Server Actions replace global state | `package.json` |

---

## Route Map

```
app/
├── (auth)/
│   ├── login/                  → /login (magic link email input)
│   └── auth/callback/          → /auth/callback (OTP exchange)
├── (app)/
│   ├── onboarding/             → /onboarding (workspace creation form)
│   └── projects/
│       └── [projectSlug]/
│           └── atcs/
│               └── [atcId]/    → /projects/:slug/atcs/:id (ATC editor)
├── invites/
│   └── accept/                 → /invites/accept?token=... (invite acceptance)
├── api/
│   ├── v1/                     → REST API (PAT or session auth)
│   └── docs/                   → Scalar API reference UI
└── qa/                         → /qa (in-app testability guide)
```

**Route groups**:
- `(auth)` — unauthenticated routes (no session required)
- `(app)` — session-protected routes (middleware enforces `PROTECTED_PREFIXES`)

---

## Styling System

| Aspect | Value | Evidence |
|--------|-------|---------|
| CSS framework | Tailwind CSS 3.4 | `package.json` devDeps |
| Token system | CSS variables mapped to Tailwind utilities | `tailwind.config.ts` |
| Dark mode | Class-based (`darkMode: ['class']`) | `tailwind.config.ts:3` |
| PostCSS | Yes (autoprefixer) | `package.json` devDeps |

### Design Token Tiers

| Token group | CSS variable pattern | Utility prefix |
|-------------|----------------------|---------------|
| Surfaces | `--bg-0` … `--bg-5` | `surface-0` … `surface-5` |
| Foreground | `--fg-0` … `--fg-4` | `fg-0` … `fg-4` |
| Strokes | `--stroke-1` … `--stroke-strong` | `stroke-1/2/3/strong` |
| Accent (vermillion) | `--accent`, `--accent-hi`, `--accent-glow`, `--accent-soft` | `accent`, `accent-hi`, etc. |
| Signal (AtcStatus) | `--pass`, `--fail`, `--blocked`, `--skipped`, `--running` + `-bg` variants | `signal-pass`, `signal-fail`, etc. |
| Layer chips (AtcLayer) | `--layer-ui`, `--layer-api`, `--layer-unit` | `layer-ui`, `layer-api`, `layer-unit` |
| Fonts | `--font-sans`, `--font-mono`, `--font-jp` | `font-sans`, `font-mono`, `font-jp` |
| Border radius | `--r-1` … `--r-4` | `rounded-1` … `rounded-4` |
| Shadows | `--shadow-pop`, `--shadow-card` | `shadow-pop`, `shadow-card` |

> Signal colors map 1:1 to `AtcStatus` enum values. Layer chip colors map to `AtcLayer` enum. Both are test-relevant: status badges and layer chips are the primary visual indicators in ATC lists.

### Typography Scale (custom — dense UI)

| Size token | px | lineHeight |
|------------|-----|------------|
| `text-2xs` | 10.5 | 1.3 |
| `text-xs` | 11 | 1.35 |
| `text-sm` | 12 | 1.4 |
| `text-base` | 13 | 1.45 |
| `text-md` | 14 | 1.45 |
| `text-lg` | 16 | 1.4 |
| `text-xl` | 18 | 1.3 |
| `text-2xl` | 22 | 1.25 |

---

## Component Library

| Library | Role | Evidence |
|---------|------|---------|
| Radix UI | Headless primitives (Dialog, DropdownMenu, Tabs, Tooltip) | `@radix-ui/*` in deps |
| shadcn/ui | Design system layered on Radix UI (component-level styling) | `tailwind.config.ts` shadcn semantic aliases |
| lucide-react | Icon set | `package.json` |
| @tanstack/react-table | ATC list tables (sorting, filtering, pagination) | `package.json` |
| @monaco-editor/react | Code editor for ATC step content (Markdown) + assertions (YAML) | `package.json` |
| cmdk | Command palette (⌘K) | `package.json` |
| sonner | Toast notifications (ATC save success/error) | `package.json` |
| @scalar/api-reference-react | Interactive API docs at `/api/docs` | `package.json` |
| class-variance-authority | Component variant API (shadcn pattern) | `package.json` |
| tailwind-merge | Conflict-free Tailwind class merging | `package.json` |

---

## Test IDs Strategy

| Aspect | State |
|--------|-------|
| `data-testid` attributes | **NOT FOUND** — no `data-testid` pattern in source |
| `aria-label` usage | Unknown — not verified in source |
| `role` attributes | Radix UI injects ARIA roles on primitives |
| Testability guide | `/qa` route in app — in-app guide for QA engineers |

> **Discovery Gap (HIGH)**: No `data-testid` strategy defined. Playwright selectors will rely on ARIA roles, visible text, and structural patterns. This is fragile for dense UI components (ATC table rows, layer chips, status badges). Recommend establishing a `data-testid` convention before automating.

---

## Key User Flows (Frontend)

```
J1 — Signup + Onboarding
  /login → email input → magic link email → /auth/callback → /onboarding → workspace form → /projects/...

J2 — ATC Authoring
  /projects/[slug]/atcs/[id] → Monaco editor (steps Markdown + assertions YAML) → save (Server Action) → toast

J3 — Team Invite
  Settings → invite form → admin POST /api/v1/workspaces/[id]/invites → email → /invites/accept?token=...

J4 — Invite Accept
  /invites/accept?token=... → token validation → membership created → redirect to workspace

J5 — API Access
  Settings → PAT form → POST /api/v1/tokens → token shown once → copy → use in API clients
```

---

## Frontend Quality

| Tool | Config | Command |
|------|--------|---------|
| ESLint | `@antfu/eslint-config` + `@next/eslint-plugin-next` | `bun run lint:check` |
| Prettier | `.prettierrc` (assumed) | `bun run format:check` |
| TypeScript | `strict: true`, `typedRoutes: true` | `bun run types:check` |
| Pre-commit | Husky + lint-staged (ESLint on `.tsx`) | On `git commit` |

---

## Discovery Gaps

| Gap | Severity | Question |
|-----|----------|---------|
| No `data-testid` strategy | HIGH | How do we locate ATC rows, status badges, layer chips reliably in Playwright? |
| ATC editor component (Monaco) | MEDIUM | What events fire on step/assertion save? Is there a save button or auto-save? |
| No `not-found.tsx` confirmed | MEDIUM | Does the app have a 404 page? |
| No `error.tsx` confirmed | HIGH | Unhandled server errors may show blank screen in production |
| Form validation UX | MEDIUM | Are Zod validation errors shown inline or via toast? |
| `/qa` route content | LOW | What testability metadata does the `/qa` page expose? |
