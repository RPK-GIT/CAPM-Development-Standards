# AI-REVIEW — Review Rules for Claude Code

Authority: **ORG** (all rules). Runtime: Both. Status: Active.
These rules govern Claude Code when it **reviews** a CAP application against this standard. The procedure that applies them is [reviews/review-model.md](../../reviews/review-model.md); the output format is [templates/review-report-template.md](../../templates/review-report-template.md).

---

## AI-REVIEW-001 — Review the actual repository, not an idea of it
**Severity: Critical.** Every finding MUST be grounded in inspected repository content (or executed checks). Claude MUST NOT report findings based on what CAP projects "usually" contain, and MUST NOT extrapolate from file names without opening the files that findings depend on.

## AI-REVIEW-002 — Determine applicability before judging
**Severity: High.** Before evaluating, Claude MUST establish the project profile — runtime (Node.js/Java), CAP versions in the lockfile/POM, single- vs multitenant, deployment target (CF/Kyma), protocols, databases — and mark rules that don't apply as `NOT APPLICABLE` with the reason. Judging a single-tenant app against `CAP-MT` rules is itself a review defect.

## AI-REVIEW-003 — Verdicts are per-rule and evidence-backed
**Severity: Critical.** Each applicable rule receives exactly one verdict: `PASS`, `FAIL`, `NOT APPLICABLE`, or `NOT ASSESSABLE` (evidence unavailable from the repository — say what would be needed). Every `PASS` and `FAIL` MUST cite the evidence: file paths and locations where possible, or the checked absence ("no `@requires` on any service in `srv/`").

## AI-REVIEW-004 — Cite rule IDs and severities from the catalog
**Severity: High.** Findings MUST cite the rule ID and use the rule's catalog severity (adjusted only downward with justification, e.g., mitigating context). Findings with no corresponding catalog rule are reported separately as **observations** labelled `AI-REC` — they MUST NOT be presented as standard violations.

## AI-REVIEW-005 — Defects and recommendations are separated
**Severity: High.** The report MUST distinguish **defects** (FAIL against a rule) from **recommendations** (improvements, observations, `AI-REC` items). A reader MUST be able to see at a glance what blocks the milestone versus what is advice.

## AI-REVIEW-006 — Never claim SAP requires something without evidence
**Severity: Critical.** A finding may state "SAP requires/recommends X" only if the cited rule carries `SAP-REQ`/`SAP-REC` authority with a working reference. Otherwise the finding is phrased as org policy or recommendation. This mirrors [authority levels](../../docs/authority-levels.md) rule 1: default down.

## AI-REVIEW-007 — Actively look for the absent
**Severity: High.** A review MUST check for missing artifacts, not only defects in present ones: missing tests for existing behavior, missing authorization on exposed services, missing error handling, missing production configuration, missing audit/privacy annotations for personal data. Absence findings cite the rule and the location where the artifact was expected.

## AI-REVIEW-008 — Security, architecture, and production-readiness get explicit sections
**Severity: Medium.** Regardless of scope, the report MUST contain explicit assessments for security concerns, architecture concerns, and production-readiness concerns (each possibly "none found"), so their absence from a report is always a deliberate statement, not an oversight.

## AI-REVIEW-009 — Independence from the implementer
**Severity: High.** When reviewing work that Claude itself (or another AI session) implemented, the review MUST be performed from the repository evidence alone — prior session claims, completion reports, and commit messages are inputs to *check*, not evidence of compliance.

## AI-REVIEW-010 — Uncertainty is reported as uncertainty
**Severity: High.** Where the reviewer cannot determine compliance (needs runtime behavior, external configuration, credentials, or SAP guidance is ambiguous), the verdict is `NOT ASSESSABLE` with the reason and what evidence would resolve it — never a guessed PASS/FAIL.

## AI-REVIEW-011 — Reproducible detection
**Severity: Medium.** For each FAIL, the report SHOULD record how the violation was detected (file inspected, pattern searched, config read) precisely enough that a human can reproduce the finding without rerunning the AI.

## AI-REVIEW-012 — No remediation during review
**Severity: High.** A standard review is read-only. Claude MUST NOT modify the repository during a review. Remediation happens afterwards, as development work under [AI-DEV rules](ai-development-rules.md), followed by re-review of the failed rules.
