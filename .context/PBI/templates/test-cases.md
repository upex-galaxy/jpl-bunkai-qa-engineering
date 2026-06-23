# Test Case Format Reference — Bunkai TMS (BK)

> Format reference only. Test Case issues (type `Test`) are authored by QA in Jira. Per `jira-required.yaml`: `sync: never` — these are NOT synced locally by `bun run jira:sync-issues`. Manage via `acli` or Xray CLI (`bun xray`).

---

## Issue Fields

| Field | Jira Name | Type | Authored by |
|-------|-----------|------|-------------|
| Summary | Summary | string | QA |
| Description | Description | rich text | QA (TC body — see format below) |
| Assignee | Assignee | user | QA |
| Test Status | Test Status 🧪 | option | QA (execution result) |
| To Be Automated | To Be Automated (QA) 🧪 | option | QA (yes \| no) |

> Local dir: `tests/` (standalone TC issues — only synced when linked to a coverable parent)

---

## Test Case Body Format

Written in the `Description` field. KATA convention: one TC = one ATC (Acceptance Test Case) or a manual test scenario not yet automated.

```markdown
## Test Case

**ID**: BK-<N>
**Title**: <descriptive title>
**Layer**: UI | API | Unit
**Type**: Functional | Security | Performance | Regression
**Priority**: P0 | P1 | P2
**ATC Reference**: ATC-<slug> (if automated counterpart exists)

### Linked Story + AC
- Story: BK-<STORY>
- AC: AC-<N> — <acceptance criterion text>

### Preconditions
- User is authenticated with role: <role>
- Workspace exists with slug: <slug>
- <any other required state>

### Test Data
| Field | Value |
|-------|-------|
| email | `<from .env>` |
| workspace_slug | `<test-slug>` |
| atc_title | `<test-title>` |

### Steps

| # | Action | Expected Result |
|---|--------|----------------|
| 1 | <action> | <expected> |
| 2 | <action> | <expected> |
| 3 | <action> | <expected> |

### Pass Criteria
- <explicit pass condition>

### Fail Criteria
- <explicit fail condition>

### Notes
<edge cases, known flakiness, environment dependencies>
```

---

## TC Lifecycle

```
draft
  → [start_design]  → in_design → [ready_to_run] → ready
  → [automation_review_from_ready] → in_review
      → [approve_to_automate]  → candidate → [start_automation] → in_automation
                                                                 → [create_pr] → pull_request
                                                                               → [merged] → automated ✓
      → [for_manual]           → manual ✓
  → [automation_review_from_draft] → in_review (skip in_design)

manual → [automation_review_from_manual] → in_review → candidate → ... → automated

deprecated (global transition from any; recover → draft)
```

---

## Jira Queries for Test Cases

```bash
# All TC issues in BK
acli issue list --jql "project = BK AND issuetype = Test"

# TCs in ready state (candidates for automation review)
acli issue list --jql "project = BK AND issuetype = Test AND status = Ready"

# TCs linked to a specific story
acli issue list --jql "issue in linkedIssues('BK-<STORY>', 'is tested by') AND issuetype = Test"

# Automation candidates
acli issue list --jql "project = BK AND issuetype = Test AND status = Candidate"

# Already automated
acli issue list --jql "project = BK AND issuetype = Test AND status = Automated"

# Manual-only verdict
acli issue list --jql "project = BK AND issuetype = Test AND status = Manual"
```

---

## Linking TCs to Stories

```bash
# Link TC to its parent story (Test link type)
acli link create --from BK-<STORY> --to BK-<TC> --type test
# Note: acli link direction inversion — "from" = STORY, "to" = TC
# Result: story "is tested by" TC; TC "tests" story

# Xray: add TC to a Test Plan
bun xray test-plan add-tests --plan BK-<TEST-PLAN> --tests BK-<TC>
```

---

## TC vs ATC — Key Distinction

| Concept | Where it lives | Created by |
|---------|---------------|------------|
| **TC** (Test Case) | Jira issue (type `Test`) | QA via `acli` / Jira UI |
| **ATC** (Acceptance Test Case) | Bunkai TMS app (`atcs` table) | QA via Bunkai ATC editor |
| Relationship | TC is the Jira traceability artifact; ATC is the executable implementation | — |
| Automation | TC transitions to `automated`; the KATA code IS the ATC | `/test-automation` |

When a TC is automated:
1. KATA ATC created in `tests/components/` (code)
2. TC in Jira transitions `→ automated`
3. `To Be Automated` field set to `yes`
4. TC linked to story via `Test` link type
5. `kata-manifest.json` updated (`bun run kata:manifest`)

---

## Bunkai-Specific TC Coverage Areas

| Area | TC Priority | Technique | Notes |
|------|------------|-----------|-------|
| RLS isolation (cross-workspace) | P0 | EP | viewer/member in WS-A cannot see WS-B data |
| ATC save — anchoring moat | P0 | EP + BVA | acIds.length = 0 → rejected; ≥1 → ok |
| Workspace bootstrap slug | P0 | BVA | length 3/40 valid; 2/41 invalid; leading-hyphen invalid |
| Role-based write block | P0 | Decision Table | role × action matrix |
| Invite lifecycle | P0 | State-Transition | pending/expired/accepted/revoked |
| PAT scope enforcement | P1 | EP | `atc:read` token → write → 403 |
| ATC fulltext search | P1 | EP | keyword match; cross-workspace = 0 results |
| AtcStatus transitions | — | State-Transition | **BLOCKED** — `runs` entity not implemented |
