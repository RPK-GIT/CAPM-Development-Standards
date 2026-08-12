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
| **Review mode** | DEVELOPMENT / RETROSPECTIVE — per [project-profile.md review modes](../development/project-profile.md); when the profile milestone is `LIVE`, append: "deployed application — findings represent current operational risk" |

## 1. Project profile
Runtime, CAP versions (from lockfile/POM), Node/JDK versions, databases per profile, protocols,
tenancy model, deployment target, auth configuration per profile, test setup.
(Basis for applicability decisions — AI-REVIEW-002.)

## 2. Verdict summary

| Category | PASS | FAIL | NOT APPLICABLE | NOT ASSESSABLE |
|---|---|---|---|---|
| CAP-XXX | n | n | n | n |
| **Total** | n | n | n | n |

**Gate recommendation:** PASS / PASS WITH EXCEPTIONS / FAIL / NOT READY / NOT APPLICABLE
(per the [milestone gate-result model](../development/rule-milestone-matrix.md#13-milestone-gate-results); uncovered HARD-GATE violation ⇒ FAIL; missing required evidence ⇒ NOT READY). ORG-PENDING rule findings are labeled as such (matrix §1.6).

## 3. Defects (rule violations)
Ordered by severity. One block per FAIL. **Finding-consolidation convention:** when multiple
rules identify the same underlying implementation defect, write ONE root-finding block instead
of repeating it per rule — heading `### Root finding: <root cause> — FAIL — <highest severity>`,
first line `**Affected rules:** <ID (severity/gate)>, <ID (severity/gate)>, …` — keeping every
rule's own evidence inside the block and every per-rule verdict in §8. The defect counts **once**
in §2's executive summary (per-rule verdict counts stay per-rule and say so). Never present one
defect as several unrelated findings.

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
Includes **cross-cutting security observations**: strong evidence of a security issue owned by a
*later* milestone's rule. Each records: owning rule ID · owning milestone · evidence (file:line) ·
risk · immediate-remediation recommendation where warranted · status (new / carried forward from
<report ref>). They do not gate this milestone (matrix §1.1) but MUST be carried into every
subsequent review until the owning milestone evaluates them.
### 5.2 Architecture concerns      (findings or "none found")
### 5.3 Production-readiness concerns (findings or "none found")
### 5.4 Missing tests              (behavior lacking coverage, per location)

## 6. NOT ASSESSABLE rules
Rule ID, why the evidence was unavailable, and what would resolve it (run tests, provide
config, deploy access, …).

## 7. Applicability decisions
Rules marked NOT APPLICABLE with one-line reasons (e.g., "CAP-MT-*: single-tenant per
profile §1"). In RETROSPECTIVE mode additionally: the SUPPORTING selection rationale —
each checklist supporting row listed as selected/not selected with the criterion
((a) profile capability / (b) evidence artifacts exist / (c) related PRIMARY finding).

## 8. Full per-rule results
Complete table: Rule ID | Verdict | Evidence pointer. (PASS verdicts also carry evidence.)

## 9. Exceptions honored
Documented exceptions (AI-DOC-002) encountered and accepted during this review, with references
(scope and expiry verified). ORG-PENDING rule findings listed with their governance status.

## 10. Standards & CAPire evidence verification
Per the [CAPire verification protocol](../reviews/capire-verification.md) — only sources relevant
to the evaluated rules; one fetch per unique URL. Mandatory traceability:
rule → project evidence → CAPire source → current source status → verdict.

| Rule | Verdict | Evidence | CAPire URL | Source status | Verified (level, timestamp) |
|---|---|---|---|---|---|
| CAP-XXX-000 | PASS | file:line | https://cap.cloud.sap/docs/… | CURRENT | L2, YYYY-MM-DD |

Governance flags (source materially changed → standards re-validation recommended): list or "none".

## 11. Remediation plan
For FAIL findings: per [remediation-plan-template.md](remediation-plan-template.md) (inline or linked).
No fixes were applied during this review (AI-REVIEW-012).

## 12. Outstanding risks & next-milestone readiness
Unresolved risks (incl. NOT-ASSESSABLE areas and soft-gate justifications); whether the entry
criteria of the next milestone are met.
```
