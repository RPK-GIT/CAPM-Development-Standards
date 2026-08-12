# CLAUDE.md — Operating Instructions for Claude Code

This repository is **CAPM Development Standards v1.0**. Claude Code interacts with it in **three distinct operating modes**. Determine the mode first; it changes what you are allowed to do.

The three modes and their command bindings:

| Mode | Command / Trigger | Reference |
|---|---|---|
| **1 — DEVELOP** an application against this standard | `/capm-develop` | [.claude/commands/capm-develop.md](.claude/commands/capm-develop.md) · [developer-guide](docs/developer-guide.md) |
| **2 — REVIEW** an application against this standard (read-only) | `/capm-review-milestone <Mx>` | [.claude/commands/capm-review-milestone.md](.claude/commands/capm-review-milestone.md) · [reviewer-guide](docs/reviewer-guide.md) |
| **3 — MAINTAIN** this standard | Editing rules / matrix / templates / commands in *this* repository | [rule-governance](docs/rule-governance.md) · [re-verification-cadence](docs/re-verification-cadence.md) |

Do not mix modes. A prompt that says *"review this project and also fix what you find"* is a Mode-2 review that must produce a report + remediation plan, then STOP — remediation happens in a separate Mode-1 session.

---

## Mode 1 — DEVELOP a CAP application against this standard

**Command:** [`/capm-develop`](.claude/commands/capm-develop.md). Binds the [development-model](development/development-model.md), the [rule-milestone matrix](development/rule-milestone-matrix.md), the project's [profile](development/project-profile.md), and the Layer-3 rules in [standards/ai/](standards/ai/README.md). Follow the [developer-guide](docs/developer-guide.md) for the practitioner's view.

### What Claude MUST do

1. **Load the project profile** (`capm-profile.yaml` at the project root). Validate per [project-profile.md](development/project-profile.md); do NOT guess missing values — propose only.
2. **Load applicable rules** — the milestone's PRIMARY + FINAL-GATE + SUPPORTING rows per [command step 5](.claude/commands/capm-develop.md), filtered by runtime + capability flags + CAP version. Never all 134.
3. **Understand HARD constraints** — HARD-gate rules block the milestone unless a valid [exception](docs/exception-governance.md) exists. Design for them from the start; do not defer.
4. **Inspect existing implementation** before proposing anything (AI-DEV-001).
5. **Prefer CAP-native mechanisms** (AI-DEV-004) — annotations before handlers, generic providers before custom CRUD, `managed` before manual timestamps.
6. **Implement incrementally**; add or update tests with the code (AI-DEV-013).
7. **Self-validate** against applicable rules; produce a completion report ([template](templates/completion-report-template.md)).

### What Claude MUST NOT do — no exceptions without explicit human approval

- Major architectural changes (AI-DEV-010) — STOP and produce an architectural-deviation report.
- New frameworks/abstractions duplicating CAP capabilities (AI-DEV-011).
- Bypass CAP abstractions (AI-DEV-012).
- Weaken security posture (AI-DEV-015).
- Remove or disable tests to make them pass (AI-DEV-014).
- Silently ignore a standard (AI-DEV-016).
- Claim compliance without evidence (AI-DEV-017).

Full list: [standards/ai/ai-development-rules.md](standards/ai/ai-development-rules.md).

---

## Mode 2 — REVIEW a CAP application against this standard

**Command:** [`/capm-review-milestone <Mx>`](.claude/commands/capm-review-milestone.md). Also invoked when the user asks *"review this project against the CAPM Engineering Standard"*. Follow the [review-model](reviews/review-model.md), the [reviewer-guide](docs/reviewer-guide.md), and the `AI-REVIEW-*` rules exactly.

### What Claude MUST do

1. **Operate read-only.** Never modify application code (AI-REVIEW-012). If remediation is needed, produce a remediation plan; do not apply it.
2. **Select applicable rules** — profile-driven, matrix-filtered, mode-aware (DEVELOPMENT vs RETROSPECTIVE). Bounded to the milestone checklist's supporting list. Never all 134.
3. **Collect evidence** from the actual repository — file:line where possible (AI-REVIEW-001). Developer assertions are not evidence unless a rule explicitly accepts a documented decision.
4. **Verify CAPire sources** at report time per [capire-verification.md](reviews/capire-verification.md). Never silently reuse prior verification.
5. **Evaluate each rule** → exactly one verdict: `PASS` / `FAIL` / `NOT APPLICABLE` / `NOT ASSESSABLE`. Never impressionistic; never convert uncertainty into PASS (AI-REVIEW-010).
6. **Give Critical rules prominence** — every applicable Critical rule gets its own report subsection with the full evidence chain (AI-REVIEW-005).
7. **Consolidate root findings** per the [report template §3 convention](templates/review-report-template.md) — multiple rules pointing to the same root cause become one block; per-rule verdicts still stand in §8.
8. **Record cross-cutting security observations** for evidence of issues owned by a later milestone's rule ([review command step 9a](.claude/commands/capm-review-milestone.md)) — carry them forward until the owning milestone evaluates them.
9. **Apply gates** per [matrix §1.3](development/rule-milestone-matrix.md#13-milestone-gate-results). HARD gate failures block unless a verified [exception](docs/exception-governance.md) is on file.
10. **Produce a report** per the [review-report template](templates/review-report-template.md) — with the mandatory traceability chain: rule → project evidence → CAPire source → source status → verdict.
11. **Never claim SAP requires something** unless the rule is SAP-REQ with a verified source (AI-REVIEW-006). Never present ORG-PENDING findings as SAP requirements (AI-DOC-004).

### What Claude MUST NOT do

- Modify the application (AI-REVIEW-012).
- Approve exceptions or create/edit exception records (that is a human role — see [exception-governance](docs/exception-governance.md)).
- Extrapolate PASS to rules not evaluated.
- Present a report that omits any template section — inapplicable sections state why.

---

## Mode 3 — MAINTAIN this standard

Applies when Claude is editing files in *this* repository: rules under `standards/`, the matrix, checklists, commands, templates, docs, or references.

### What Claude MUST do

1. **Verify SAP evidence live** for every load-bearing change. `L2` explicit fetch minimum for security/privacy/tenant/Critical rule changes; `L3` for version-sensitive; the [CAPire protocol](reviews/capire-verification.md) is the authority.
2. **Preserve authority.** Never silently convert `SAP-REC` → `SAP-REQ` or the reverse. See [rule-governance.md § Authority assignment](docs/rule-governance.md#3-authority-assignment).
3. **Maintain rule IDs.** Stable; never renumbered; never reused. Deprecated/retired rules stay on file. See [rule-governance § 11](docs/rule-governance.md#11-rule-ids--stable-and-never-reused).
4. **Update the version register** ([docs/version-management.md](docs/version-management.md)) when CAP releases. See [re-verification-cadence.md](docs/re-verification-cadence.md).
5. **Update mappings** when a rule's milestone/gate/condition changes. Verify the matrix still has exactly 134 rule rows in §2.
6. **Update validation scenarios** ([docs/operational-validation.md](docs/operational-validation.md)) affected by the change, and execute them — do not claim all-scenarios-pass without evidence.
7. **Produce change records** in [CHANGELOG.md](CHANGELOG.md) — human-readable, with rule IDs and one-line summaries.
8. **Bump `Last verified` dates** on rules touched.
9. **Preserve history.** No silent rewrites of normative text; amendments carry inline notes or matrix footnotes with the change date and evidence reference.

### What Claude MUST NOT do

- Add application code, build tooling, or runtime dependencies to this repository — it is documentation-only.
- Add rules without SAP evidence (or explicit GEN/ORG/AI-REC labeling and rationale). See [rule-governance § 1–3](docs/rule-governance.md).
- Commit secrets, credentials, `.env` files, tokens, hosts, customer database IDs, or destination names.
- Ratify ORG-PENDING rules — that is an organizational decision, not a Claude decision (see [org-ratification.md](docs/org-ratification.md)).
- Approve exception records — see Mode 2.
- Amend historical commits or rewrite pilot history.

---

## Quick navigation

| Need | Read |
|---|---|
| Executive overview | [docs/executive-overview.md](docs/executive-overview.md) |
| The standard's design | [docs/standard-architecture.md](docs/standard-architecture.md) |
| Authority taxonomy | [docs/authority-levels.md](docs/authority-levels.md) |
| Version policy | [docs/version-management.md](docs/version-management.md) |
| Principles (Layer 1) | [standards/principles/engineering-principles.md](standards/principles/engineering-principles.md) |
| Rule catalog (Layer 2) | [standards/rules/README.md](standards/rules/README.md) |
| AI rules (Layer 3) | [standards/ai/README.md](standards/ai/README.md) |
| Milestones & gates | [development/lifecycle.md](development/lifecycle.md) · [rule-milestone matrix](development/rule-milestone-matrix.md) · [per-milestone checklists](development/milestones/m0-requirements.md) |
| SAP source map | [references/sap-cap-sources.md](references/sap-cap-sources.md) |
| Project profile & commands | [development/project-profile.md](development/project-profile.md) · [.claude/commands/](.claude/commands/capm-develop.md) · [operational validation](docs/operational-validation.md) |
| Governance | [rule-governance](docs/rule-governance.md) · [exception-governance](docs/exception-governance.md) · [org-ratification](docs/org-ratification.md) · [re-verification-cadence](docs/re-verification-cadence.md) |
| Adoption | [adoption-guide](docs/adoption-guide.md) · [developer-guide](docs/developer-guide.md) · [reviewer-guide](docs/reviewer-guide.md) · [adoption-boundaries](docs/adoption-boundaries.md) |
| Pilot history | [pilots/pilot-1-scs-m3/README.md](pilots/pilot-1-scs-m3/README.md) · [Round 1 calibration](pilots/round-1-calibration.md) · [Round 2](pilots/pilot-1-scs-m3-round-2/README.md) · [Round 3 remediation](pilots/pilot-1-scs-m3-round-3/README.md) |

## Adoption in a CAP project

Adopting projects: copy [templates/capm-profile-template.yaml](templates/capm-profile-template.yaml) to the project root as `capm-profile.yaml` (fill it at M0), make this standards repository available (vendored, submodule, or sibling checkout) and reference its version (`v1.0.0` or a pinned commit) in `standard.version`, and copy the two command files into the project's `.claude/commands/`. Development evidence lands in the project under `docs/adr/` and `docs/capm/` for the review workflow to consume. Ten-step guide: [docs/adoption-guide.md](docs/adoption-guide.md). Worked examples (ILLUSTRATIVE / NON-PRODUCTION): [examples/](examples/worked-example-m3/README.md). Real-project pilot: [pilots/](pilots/pilot-1-scs-m3/README.md).
