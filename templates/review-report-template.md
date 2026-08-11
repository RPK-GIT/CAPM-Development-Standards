# Review Report Template

Canonical output format for CAPM Standard reviews ([review model](../reviews/review-model.md)). Sections are never dropped; inapplicable sections state why.

```markdown
# CAPM Standard Review Report

| Field | Value |
|---|---|
| **Project** | name / repository |
| **Reviewed revision** | commit SHA / branch |
| **Review date** | YYYY-MM-DD |
| **Reviewer** | Claude Code (model/version) / human |
| **Standard version** | CAPM Engineering Standard @ <commit/tag> |
| **Scope** | full / milestone Mx / categories |
| **Review type** | initial / re-review of <report ref> |

## 1. Project profile
Runtime, CAP versions (from lockfile/POM), Node/JDK versions, databases per profile, protocols,
tenancy model, deployment target, auth configuration per profile, test setup.
(Basis for applicability decisions — AI-REVIEW-002.)

## 2. Verdict summary

| Category | PASS | FAIL | NOT APPLICABLE | NOT ASSESSABLE |
|---|---|---|---|---|
| CAP-XXX | n | n | n | n |
| **Total** | n | n | n | n |

**Gate recommendation:** PASS / NOT PASSED — with the blocking findings listed.
(Critical or unexcepted High ⇒ NOT PASSED.)

## 3. Defects (rule violations)
Ordered by severity. One block per FAIL:

### <RULE-ID> — <Rule title> — FAIL — <Severity>
- **Authority:** SAP-REQ / SAP-REC / GEN / ORG (from the rule; phrase "SAP requires" only for SAP-REQ)
- **Evidence:** file:line reference(s) or verified absence ("no X found in Y")
- **Detection:** how found (files inspected / patterns searched / config read) — reproducible
- **Impact:** consequence of the violation in this project
- **Remediation:** concrete, CAP-native-first suggestion (advisory — remediation is separate work)

## 4. Recommendations & observations (non-defects)
Improvements without a violated catalog rule. Each labelled `AI-REC` (or the rule's
authority if tied to a Low-severity SHOULD), with evidence. Never mixed with Section 3.

## 5. Cross-cutting assessments
### 5.1 Security concerns          (findings or "none found", with what was checked)
### 5.2 Architecture concerns      (findings or "none found")
### 5.3 Production-readiness concerns (findings or "none found")
### 5.4 Missing tests              (behavior lacking coverage, per location)

## 6. NOT ASSESSABLE rules
Rule ID, why the evidence was unavailable, and what would resolve it (run tests, provide
config, deploy access, …).

## 7. Applicability decisions
Rules marked NOT APPLICABLE with one-line reasons (e.g., "CAP-MT-*: single-tenant per
profile §1").

## 8. Full per-rule results
Complete table: Rule ID | Verdict | Evidence pointer. (PASS verdicts also carry evidence.)

## 9. Exceptions honored
Documented exceptions (AI-DOC-002) encountered and accepted during this review, with references.
```
