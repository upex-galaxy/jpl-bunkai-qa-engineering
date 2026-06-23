# Test Plan Format Reference — Bunkai TMS (BK)

> Format reference only. Covers two plan levels: Story ATP (Acceptance Test Plan on the Story) and Epic Feature Test Plan. Both are authored by QA, pushed to Jira fields, then synced locally. Read synced files — never hand-write them.

---

## Plan Levels

| Level | Issue Type | Field | Syncs to |
|-------|-----------|-------|---------|
| Story ATP | Story | `customfield_10067` (`🧪 Acceptance Test Plan (ATP)`) | `acceptance-test-plan.md` |
| Epic Feature Test Plan | Epic | Feature Test Plan (QA) field | `feature-test-plan.md` |
| Xray Test Plan (Modality Xray) | Test Plan | Issue description | `.context/PBI/test-plans/TESTPLAN-BK-<N>-<slug>/` |

---

## Story-Level ATP

Written by QA after shift-left review. Pushed to Jira via `acli field update` or as a comment fallback. Authoritative once synced.

```markdown
# Acceptance Test Plan — BK-<N>: <Story Summary>

## Meta
- **Sprint**: <sprint name>
- **QA Engineer**: <name>
- **Test Environment**: local | staging | production
- **App URL**: http://localhost:3000 | <staging-url>
- **Test accounts**: see `.env` keys LOCAL_USER_EMAIL / STAGING_USER_EMAIL

## Feature Understanding
<1–3 sentences explaining what the feature does and how it works>

## Test Scope

### In Scope
- <feature area 1>
- <feature area 2>

### Out of Scope
- <exclusion 1>

## Test Cases

### P0 — Smoke / Critical Path

| TC | Title | Layer | AC | Technique |
|----|-------|-------|----|-----------|
| BK-<N> | <title> | UI \| API \| Unit | AC-1 | EP |

### P1 — Edge Cases + Validation

| TC | Title | Layer | AC | Technique |
|----|-------|-------|----|-----------|
| BK-<N> | <title> | API | AC-2 | BVA |

### P2 — Non-Critical / Exploratory

| TC | Title | Type |
|----|-------|------|
| — | <exploratory-scenario> | Exploratory |

## Test Design Notes
<EP/BVA/State-Transition choices and rationale>

## Entry Criteria
- [ ] Story in "Ready For QA"
- [ ] Build deployed to <env>
- [ ] Test accounts configured in .env
- [ ] No P0 blockers from previous sprint

## Exit Criteria
- [ ] All P0 TCs executed (PASS or documented FAIL)
- [ ] All P1 TCs executed
- [ ] Open bugs triaged with severity
- [ ] ATR written and pushed to Jira

## Risks
| Risk | Mitigation |
|------|-----------|
| ATC status transition not implemented | Test AtcStatus display only; skip execution tests |
| Staging URL unknown | Execute against local only until URL provided |
```

---

## Epic-Level Feature Test Plan

Written once per Epic. Lives on the Epic's `feature_test_plan` field. Syncs to `feature-test-plan.md`.

```markdown
# Feature Test Plan — BK-EPIC-<N>: <Epic Summary>

## Coverage Summary

| Story | Key | Test Focus | Priority |
|-------|-----|-----------|---------|
| Workspace Bootstrap | BK-<N> | FR-001 — slug/name validation, RPC atomicity, RLS | P0 |
| ATC Authoring | BK-<N> | FR-004 — anchoring moat, layer, steps/assertions | P0 |
| Team Invites | BK-<N> | FR-002/003 — token lifecycle, expiry, email match | P0 |
| PAT Auth | BK-<N> | FR-005 — scope enforcement, hash-only storage | P1 |

## Risk-Based Coverage Rationale
<why certain stories have elevated test depth — RLS, anchoring moat, security>

## Automation Candidates
| Story | TC Count | Automate? | Why |
|-------|---------|-----------|-----|
| BK-<N> | 8 | Yes (P0) | Regression risk — RLS + auth flows |
| BK-<N> | 3 | Deferred | ATC status flow blocked on `runs` entity |

## Cross-Story Risks
- ATC status transition mechanism missing (CRITICAL) — blocks execution testing
- No `data-testid` convention — Playwright selectors fragile on dense UI
- Staging URL unknown — all E2E tests blocked until URL provided

## Dependencies
- `runs` / test execution entity (future migration) — blocks AtcStatus transition tests
- `/adapt-framework` completion — required before any automated tests can run
```

---

## Xray Test Plan (Modality Xray)

If the workspace uses Xray (`tms_cli: bun xray`), the ATP lives in a dedicated **Test Plan** issue (type `Test Plan`, key `BK-<N>`). Its description holds the ATP body. Linked to the Story via `Test` link type.

```bash
# Create a Test Plan issue
bun xray test-plan create --project BK --summary "ATP: BK-<STORY>: <story-summary>"

# Link to the story
acli link create --from BK-<STORY> --to BK-<TEST-PLAN> --type test

# Add Test Cases to the Test Plan
bun xray test-plan add-tests --plan BK-<TEST-PLAN> --tests BK-<TC1>,BK-<TC2>

# Create a Test Execution under the Test Plan
bun xray test-execution create --plan BK-<TEST-PLAN> --summary "ATR: Sprint <N> Execution"
```

---

## Test Design Technique Triggers (binding)

Per `agentic-qa-core/references/test-design-doctrine.md`:

| Trigger | Technique | Bunkai example |
|---------|-----------|----------------|
| Any condition | Equivalence Partitioning (EP) | valid slug / invalid slug |
| Range / limit field | Boundary Value Analysis (BVA) | slug length 3–40; acIds.length ≥ 1 |
| Status field | State-Transition | AtcStatus, MemberStatus, WorkspaceInvite lifecycle |
| 2+ interacting conditions | Decision Table | role × action (viewer write → 403, member write → OK) |
| 3+ factors | Pairwise | layer × status × acIds count |
| Known failure modes | Error Guessing | empty title, expired token, duplicate slug |

ACs are the **floor** — cover all ACs AND risk beyond them. One AC → multiple test cases (1:N default). Collapse only with `trivially atomic` written justification.
