# Bug Report Format Reference — Bunkai TMS (BK)

> Format reference only. Bug content is synced from Jira via `bun run jira:sync-issues get BK-<N>`. Never author `[SYNC]` files locally. Push bug data to Jira fields, then sync.

---

## Issue Fields

| Field | Jira Name | Custom Field | Type | Values |
|-------|-----------|-------------|------|--------|
| Summary | Summary | (standard) | string | `[BK-<N>] <short description>` |
| Description | Description | (standard) | rich text | Steps to reproduce |
| Priority | Priority | (standard) | option | Blocker / Critical / Major / Minor / Trivial |
| Assignee | Assignee | (standard) | user | Dev assigned to fix |
| Affected Version | Affects Version | (standard) | version | Build where bug found |
| Fix Version | Fix Version | (standard) | version | Build where fix lands |
| Actual Result | 🐞 Actual Result | `customfield_10056` | string | Observed behavior |
| Expected Result | ✅ Expected Result | (see fields.json) | string | Expected per AC/spec |
| Error Type | Error Type | (see fields.json) | option | See values below |
| Severity | Severity 🚩 | (see fields.json) | option | See values below |
| Test Environment | Test Environment 📦️ | (see fields.json) | option | See values below |
| Root Cause | Root Cause 🐞 | (see fields.json) | option | See values below (set post-fix) |
| Workaround | 🚩 Workaround | (see fields.json) | string | Temporary mitigation |
| Evidence | 🧫EVIDENCE | (see fields.json) | string | Screenshot URL / video / log |
| Fix | Fix | (see fields.json) | option | bugfix \| hotfix |

---

## Controlled Field Values

### Error Type
`content` · `crash` · `data` · `functional` · `integration` · `performance` · `security` · `visual`

### Severity
| Value | Description |
|-------|-------------|
| `critica` | System crash / data loss / security breach |
| `mayor` | Core feature broken; no workaround |
| `moderada` | Feature degraded; workaround exists |
| `menor` | Minor UX issue; no functional impact |
| `trivial` | Cosmetic / typo / low-visibility |

### Test Environment
`dev` · `production` · `qa` · `staging` · `uat`

### Root Cause (set after fix)
`code_error` · `config_env_error` · `data_error` · `environment_error` · `integration_error` · `requirement_error` · `third_party_error` · `working_as_designed`

---

## Bug Report Body (Description Field — Steps to Reproduce)

```markdown
## Bug Summary
<one sentence>

## Environment
- URL: <page URL where bug occurred>
- Test Environment: local | staging | production
- Build: <commit SHA or app version>
- Browser: Chrome 126 / Firefox 128 / Safari 17
- OS: Windows 11 / macOS 15 / Ubuntu 22.04
- User Role: viewer | member | admin | owner
- Workspace: <workspace-slug>

## Steps to Reproduce
1. <step 1>
2. <step 2>
3. <step 3>

## Actual Result
<what happened>

## Expected Result
<what should have happened per AC or spec>

## Error Type
<functional | visual | data | ...>

## Severity
<critica | mayor | moderada | menor | trivial>

## Workaround
<temporary mitigation, or "None">

## Evidence
<screenshot embed or URL; video URL; browser console error; network request>
```

---

## Bug Lifecycle (QA transitions)

```
open
  ├── [start_fixing]      → in_progress → in_review → ready_for_qa → [retest_passed] → closed ✓
  ├── [is_cnr]            → cannot_reproduce ✗
  ├── [is_duplicated]     → duplicated ✗
  ├── [is_wad]            → rejected ✗
  ├── [is_not_a_bug]      → enhancement ✗
  └── [defer]             → deferred (resume via resume_fix)

From any status: [re_open] → open
```

---

## Bug Severity Mapping for Jira Priority

| Severity field | Jira Priority |
|----------------|--------------|
| `critica` | Blocker |
| `mayor` | Critical |
| `moderada` | Major |
| `menor` | Minor |
| `trivial` | Trivial |

---

## Linked Artifacts

```
BK-<BUG>
  ├── is caused by → BK-<STORY>    (Problem/Incident — story caused this bug)
  └── blocks       → BK-<STORY>    (Blocks — open bug blocks story from qa_approved)
```

---

## Bunkai-Specific Bug Taxonomy

| Feature Area | Common Error Types | Severity Hints |
|-------------|-------------------|----------------|
| RLS / authZ | security, functional | critica if cross-workspace data leak |
| ATC save (Server Action) | functional, data | mayor if AC anchoring bypassed |
| Invite flow | functional | mayor if expired/used token accepted |
| PAT auth | security | critica if wrong-scope token grants access |
| Magic link auth | security | mayor if OTP replay succeeds |
| ATC fulltext search | functional | menor if cross-workspace results appear |
| UI / status badges | visual | menor/trivial |
| Monaco editor | functional | mayor if ATC content not saved |
