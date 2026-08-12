# Review Model — Independent Standard Review by Claude Code

How Claude Code performs a review when asked, e.g.:

> “Review this project against the CAPM Engineering Standard.”

The binding rules are the [AI-REVIEW family](../standards/ai/ai-review-rules.md); the output format is [templates/review-report-template.md](../templates/review-report-template.md). A review is **read-only** (AI-REVIEW-012): no repository modifications, no fixes — findings only.

## Scope forms

| Request | Scope |
|---|---|
| Full standard review | All applicable Layer 2 categories |
| Milestone gate review (e.g., "review milestone M6") | The milestone's PRIMARY + FINAL-GATE rules (plus SUPPORTING rules selected per review mode — DEVELOPMENT: rows whose subject the milestone's changes touch; RETROSPECTIVE: evidence-driven selection per the review command) per the [rule-milestone matrix](../development/rule-milestone-matrix.md) and the milestone's [checklist](../development/milestones/m0-requirements.md); gate result per [matrix §1.3](../development/rule-milestone-matrix.md#13-milestone-gate-results) |
| Category review (e.g., "security review") | Named categories only — the report states the narrowed scope |
| Re-review after remediation | Previously failed rules + anything touched by the remediation |

## Procedure

### Step 1 — Load the standard
Read the Layer 2 category files in scope ([standards/rules/](../standards/rules/README.md)) and this procedure. Only **Active** catalog rules are citable; candidate rules from `references/` are not (AI-REVIEW-004).

### Step 2 — Profile the project *(AI-REVIEW-002)*
Establish from the repository: runtime (Node.js/Java); exact CAP dependency versions (lockfile / effective POM); databases per profile; protocols; single- vs multitenant; deployment target; auth configuration per profile; test framework; project layout. Record the profile in the report — it determines rule applicability and CAP-version applicability.

### Step 3 — Select applicable rules
For each in-scope category, mark every rule Applicable / Not applicable (with reason). Check each rule's CAP-version field against the profiled versions; where a rule doesn't apply to the project's version, mark it and say so.

### Step 4 — Gather evidence *(AI-REVIEW-001, -007)*
For each applicable rule, execute its **Detection guidance**: open the relevant files, search for the relevant patterns, read the relevant configuration. Evidence includes verified absence. Findings may not rest on unopened files. Static inspection is the default; where a rule requires dynamic verification (running tests/build), either run it or mark the rule `NOT ASSESSABLE` with what's needed (AI-REVIEW-010).

### Step 5 — Verdicts *(AI-REVIEW-003)*
Per applicable rule: `PASS` / `FAIL` / `NOT ASSESSABLE`, each with evidence (file:line where possible). FAIL findings carry the rule's severity and the rule's authority level — phrase "SAP requires…" only for `SAP-REQ` rules (AI-REVIEW-006).

### Step 6 — Cross-cutting assessments *(AI-REVIEW-008)*
Regardless of per-rule results, explicitly assess and report:
- **Security concerns** — including exposure, auth gaps, secrets, tenant isolation
- **Architecture concerns** — including CAP-abstraction bypasses, framework duplication
- **Production-readiness concerns** — including observability, deployment hygiene, operational gaps
- **Missing tests** — behavior present in code but absent from the suite

### Step 7 — Report *(AI-DOC-003)*
Produce the report per [review-report-template.md](../templates/review-report-template.md):
- Verdict summary and counts, findings ordered by severity
- Defects (rule FAILs) strictly separated from recommendations/observations (AI-REVIEW-005)
- Reproducible detection notes per FAIL (AI-REVIEW-011)
- Gate recommendation per the [milestone gate-result model](../development/rule-milestone-matrix.md#13-milestone-gate-results): uncovered HARD-GATE violation ⇒ **FAIL**; missing required evidence ⇒ **NOT READY** (consistent with the lifecycle severity gating)

## Review integrity constraints

- The reviewer trusts only repository evidence — completion reports and commit messages are claims to verify, not proof (AI-REVIEW-009).
- Uncertainty is never converted into a verdict (AI-REVIEW-010).
- The review never invents rules: concerns outside the catalog are reported as `AI-REC` observations, clearly labelled (AI-REVIEW-004).
- Remediation is a separate, subsequent development activity under the [development model](../development/development-model.md), followed by re-review of failed rules.
