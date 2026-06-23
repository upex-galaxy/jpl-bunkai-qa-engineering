# Comments for BK-93

[View in Jira](https://jira.upexgalaxy.com/browse/BK-93)

---

### Jorgelina Abdo - 8/6/2026, 6:33:07

Bug found during exploratory testing of BK-10 (TC-I04 — PAT bearer token rejected on module/workspace endpoints). Story: https://upexgalaxy69.atlassian.net/browse/BK-10

---

### Ely - 10/6/2026, 13:50:26

## 🔗 Duplicate Bug Resolution

This issue is a ***duplicate**** of ****BK-84***.

### Analysis

- Both describe: a valid PAT bearer (`bk*pat**`) rejected with `401 unauthorized` on module/workspace member-resource endpoints while /me + /workspaces work.
- Same root cause: pre-ADR-0001, ~29 handlers used the cookie-only `createClient().auth.getUser()` and ignored the Authorization header. Fixed structurally by commit `226fc9d` (unified auth gateway) and live-verified on staging 2026-06-10 — full evidence on BK-84.
- BK-84 status: Ready For QA.

### Action Taken

- Linked to BK-84 (Duplicate) and closed as Duplicated.

***Note******:*** progress and QA verification tracked on BK-84.

---

### Automation for Jira - 10/6/2026, 13:50:30

Hola Jorgelina Abdo!
Este reporte es idéntico a otro reporte de incidencia de la misma US.
Toma en cuenta que cada reporter de defecto/bug debe ser independiente y único. No podemos trabajar en más de 1 defecto cuya incidencia es la misma pero con datos diferentes. Es mejor juntar todos los defectos en un mismo reporte por cada única funcionalidad.
Por lo tanto, éste reporte se considera DUPLICADO, y para no trabajar en 2 incidencias iguales, vamos a considerar una (la primera que se creó) para trabajar cómodamente.
Para cualquier duda, escríbenos por Slack! 🕵🏻‍♂️

---


_Synced from Jira by sync-jira-issues_
