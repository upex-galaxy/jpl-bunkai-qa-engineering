# User Story Format Reference — Bunkai TMS (BK)

> Format reference only — NOT a template to fill in. Per-ticket content is synced from Jira via `bun run jira:sync-issues get BK-<N> --include-comments`. Never author these fields locally; push them to Jira, then sync.

---

## Issue Fields

| Field | Jira Name | Custom Field | Type | Authored by |
|-------|-----------|-------------|------|-------------|
| Summary | Summary | (standard) | string | PO/BA |
| Description | Description | (standard) | rich text | PO/BA |
| Story Points | Story Points | (standard) | number | Dev/QA (planning) |
| Assignee | Assignee | (standard) | user | Dev |
| Sprint | Sprint | (standard) | sprint | PM |
| Epic Link | Epic Link | (standard) | link | PM |
| Acceptance Criteria | ✅ Acceptance Criteria (Gherkin) | `customfield_10063` | string | PO |
| Business Rules | 🚩Business Rules Specification | (see fields.json) | string | PO |
| Scope | Scope ⛳ | (see fields.json) | string | PO |
| Out of Scope | 🏴 Out Of Scope | (see fields.json) | string | PO |
| Workflow | 🧬WORKFLOW | (see fields.json) | string | PO |
| Mockup | Mockup 🎴 | (see fields.json) | string | Design |
| Weblink | Weblink (URL) 🌍️ | (see fields.json) | string | PO |
| Acceptance Test Plan (ATP) | 🧪 Acceptance Test Plan (ATP) | `customfield_10067` | string | **QA** |
| Acceptance Test Results (ATR) | 🧪 Acceptance Test Results (ATR) | `customfield_10147` | string | **QA** |
| Feature Test Plan | 🧪 Feature Test Plan (QA) | (see fields.json) | string | **QA** (Epic-level) |
| Spec Implementation Plan | Spec Implementation Plan (Dev)🛠️ | (see fields.json) | string | Dev |

---

## Acceptance Criteria Format (Gherkin)

Acceptance criteria are authored by PO in `customfield_10063`. QA reads them from the synced `acceptance-criteria.md`.

```gherkin
Feature: <feature-name>

  Scenario: <AC title>
    Given <precondition>
    When <action>
    Then <expected outcome>

  Scenario: <AC 2>
    Given ...
    When ...
    Then ...
```

---

## Acceptance Test Plan (ATP) Format

QA authors the ATP and writes it to `customfield_10067` via `acli`. Modality jira-native: ATP lives on the Story field. Syncs to `.context/PBI/epics/.../stories/.../acceptance-test-plan.md`.

```markdown
## Acceptance Test Plan (ATP)

**Issue**: BK-<N>
**Story**: <summary>
**QA Engineer**: <name>
**Sprint**: <sprint-name>
**Test Environment**: local | staging | production

### Objectives
<what will be verified>

### Scope
- In scope: <list>
- Out of scope: <list>

### Test Cases

| TC-ID | Title | Type | Priority | AC | Technique |
|-------|-------|------|----------|----|-----------|
| BK-<TC1> | <title> | Manual | P0 | AC1 | EP |
| BK-<TC2> | <title> | Manual | P1 | AC2 | BVA |

### Test Data Requirements
<test accounts, seed data, environment prep>

### Entry Criteria
- Story is in "Ready For QA" status
- Build deployed to <env>
- Test accounts configured in .env

### Exit Criteria
- All P0/P1 test cases executed
- No open critical/blocker bugs
```

---

## Acceptance Test Results (ATR) Format

QA authors the ATR and writes it to `customfield_10147` via `acli`. Syncs to `acceptance-test-results.md`.

```markdown
## Acceptance Test Results (ATR)

**Issue**: BK-<N>
**Story**: <summary>
**QA Engineer**: <name>
**Sprint**: <sprint-name>
**Test Environment**: <env>
**Date**: <date>
**Build**: <commit SHA or version>

### Test Execution Summary

| TC-ID | Title | Status | Notes |
|-------|-------|--------|-------|
| BK-<TC1> | <title> | ✅ PASS | — |
| BK-<TC2> | <title> | ❌ FAIL | Linked to BK-<BUG> |
| BK-<TC3> | <title> | ⚠️ BLOCKED | <reason> |

### Bugs Found

| Bug ID | Title | Severity | Status |
|--------|-------|----------|--------|
| BK-<N> | <summary> | Mayor | Open |

### QA Verdict

**Status**: [PASS | FAIL | BLOCKED]
**Decision**: [qa_sign_off | defect_reported | back_from_in_test]
**Notes**: <free text>
```

---

## Linked Artifacts

```
BK-<STORY>
  ├── is tested by → BK-<TEST-PLAN>   (Xray ATP)
  ├── is tested by → BK-<TC-1..N>     (Test Cases)
  ├── causes       ← BK-<BUG/DEFECT>  (Problem/Incident)
  └── is blocked by ← BK-<DEFECT>     (Blocks — when defect_reported fires)
```

---

## Story Status Flow (QA perspective)

```
ready_for_qa
  → [QA picks up] → in_test
  → [All pass]    → qa_approved  (transition: qa_sign_off)
  → [Bug found]   → blocked      (transition: defect_reported)
  → [Wrong build] → ready_for_qa (transition: back_from_in_test)
```
