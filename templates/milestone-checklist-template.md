# Milestone Checklist Template

One checklist instance per project per milestone ([lifecycle](../development/lifecycle.md)). The pre-filled per-milestone checklists (concrete rule lists, gates, evidence) exist under [development/milestones/](../development/milestones/m0-requirements.md) — instantiate this template per project and work through the corresponding milestone file. Gate results follow the [matrix §1.3 model](../development/rule-milestone-matrix.md#13-milestone-gate-results).

```markdown
# Milestone Checklist — Mx: <Milestone name>

| Field | Value |
|---|---|
| **Project** | name / repository |
| **Milestone** | Mx — name |
| **Date opened / closed** | YYYY-MM-DD / YYYY-MM-DD |
| **Standard version** | CAPM Engineering Standard @ <commit/tag> |

## 1. Deliverables
- [ ] <deliverable per lifecycle.md for this milestone> — location/link

## 2. Development complete
- [ ] All in-scope requirements implemented
- [ ] Completion report(s) produced — link(s)

## 3. Self-validation complete
- [ ] Applicable rule categories checked: <CAP-XXX, …>
- [ ] Deviations declared and exceptions filed (or "none")

## 4. Tests
- [ ] Milestone-required tests exist and pass — evidence (CI run / command output)
- [ ] Unhappy paths covered where required

## 5. CAPM standard review
- [ ] Independent review performed — report link
- [ ] Verdict counts: PASS __ / FAIL __ / N-A __ / NOT ASSESSABLE __

## 6. Remediation
- [ ] All Critical findings resolved
- [ ] All High findings resolved or exception approved — references
- [ ] Medium/Low findings dispositioned (fix / accept / defer with owner)
- [ ] Re-review of failed rules passed — report link

## 7. Exit criteria (per lifecycle.md for this milestone)
- [ ] <exit criterion 1>
- [ ] <exit criterion 2>

## Sign-off
| Role | Name | Date | Decision |
|---|---|---|---|
| Reviewer | | | |
| Approver | | | PASS / NOT PASSED |
```
