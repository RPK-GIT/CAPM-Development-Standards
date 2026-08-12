# Rule Governance

How a rule enters, evolves, and leaves the [Layer 2 rule catalog](../standards/rules/README.md). Applies to the 134 catalog rules and — with the AI-family exceptions noted — to Layer 3 (`standards/ai/`).

The rules are the standard's normative core; they are the authored SAP-grounded (and, where marked, organizational or general-engineering) statements that reviews and development bind to. History matters: a reviewer must be able to reconstruct what a rule said at a given point in time.

## 1. Proposal

A change proposal is a rule ID + delta + evidence. It may come from:

- **Pilot calibration** — a real-project pilot surfaces a detection defect, mechanism-list drift, or milestone-mapping gap. Recorded as `STANDARD CHANGE PROPOSAL` in the pilot report (see [Pilot 1 Round 1 calibration](../pilots/round-1-calibration.md) — three accepted, two rejected).
- **CAPire re-verification** — the quarterly sweep ([re-verification cadence](re-verification-cadence.md)) finds a source has materially changed or been removed. Load-bearing rule text is re-verified per the [CAPire protocol](../reviews/capire-verification.md).
- **New SAP guidance** — a CAP major/minor release adds a mechanism the catalog does not yet name.
- **A reviewer or developer** — via issue/PR, using the [rule-template](../templates/rule-template.md) for new rules or an inline diff for amendments.

The proposal MUST cite:

- The specific rule ID (or "new rule").
- The current wording being changed.
- The concrete evidence (repository code path, live CAPire URL + verbatim quote, pilot finding reference).
- Expected impact on authority, severity, runtime applicability, milestone mapping, and existing validation scenarios.

**Never accept a proposal that changes a rule "to make findings disappear."** Proposals that only reduce reported friction without evidence of a rule defect are rejected on their face.

## 2. SAP evidence collection

Every SAP-technical claim must trace to an official SAP source (primarily `cap.cloud.sap/docs/`) via the [source map](../references/sap-cap-sources.md). Third-party sources are never treated as authoritative where SAP documentation exists.

- **L1** — URL liveness for orientation (`cap.cloud.sap/docs/…` responds).
- **L2** — verbatim quote of the load-bearing sentence(s) present in the current page. Required for Critical, security, privacy, tenant-isolation rules and for any wording change to the rule statement.
- **L3** — L2 + version-register cross-check ([docs/version-management.md](version-management.md)). Required for version-sensitive rules.

Evidence is recorded in the proposal AND, on acceptance, in the rule's SAP-reference field + `Last verified` date.

## 3. Authority assignment

The [authority taxonomy](authority-levels.md) is normative. Never silently convert authority levels.

| Level | Applies when |
|---|---|
| **SAP-REQ** | SAP documentation states the rule as a requirement (`MUST`, `have to`, "required", or equivalent) |
| **SAP-REC** | SAP recommends but does not require (`SHOULD`, "recommended", "prefer") |
| **GEN** | General engineering practice with no SAP-specific source, but genuinely CAP-relevant (currently: [CAP-PERF-007](../standards/rules/performance.md), [CAP-PRIV-004](../standards/rules/data-privacy.md)) |
| **ORG** | Organization-specific policy — SAP is silent. Marked `ORG-PENDING` until ratified (see [org-ratification.md](org-ratification.md)) |
| **AI-REC** | Claude's inference from convergent SAP guidance where no single source states the rule — used only in Layer 3 rules |

If a rule's authority would need to be **strengthened**, evidence must be a NEW SAP source explicitly making the requirement. If it would need to be **weakened**, evidence must be a SAP source materially changing what it says. The default for ambiguity is: keep the current authority and record the concern in [research-gaps.md](../references/research-gaps.md).

## 4. Severity assignment

Severity is **not** derived from authority. The four levels:

- **Critical** — violation directly enables unauthorized read/write, credential exposure, tenant leak, injection, or platform-integrity compromise. Requires a *Critical justification* section in the rule body.
- **High** — violation materially affects security posture, data integrity, API compatibility, or production reliability.
- **Medium** — quality, maintainability, or scoped correctness defect that does not by itself compromise the system.
- **Low** — ADVISORY-only guidance.

A proposal that would raise severity must show the failure mode. A proposal to lower severity must show it doesn't match one of the higher tiers. Severity is not adjusted for report-length management.

## 5. Runtime and version applicability

- **Runtime** — one of `Node.js`, `Java`, `Both`. Set from the SAP source's runtime scope AND from the rule's mechanism (annotation? applies to both; runtime property? one runtime).
- **CAP version** — the range the rule applies to. Prefer "All currently supported versions" where the mechanism is stable; otherwise name the minimum version and mark the rule ⏱ *version-sensitive* so [Level 3 verification](../reviews/capire-verification.md) fires.

## 6. Gate class

Gate class (HARD / SOFT / ADVISORY) is set per rule and applied at PRIMARY and FINAL-GATE milestones (matrix [§1.1](../development/rule-milestone-matrix.md#11-applicability-per-rule--milestone), [§1.2](../development/rule-milestone-matrix.md#12-gate-class-per-rule--applied-where-the-rule-is-evaluated-as-primary-or-final-gate-supporting-appearances-are-non-gating-see-11)). It is **not** derived mechanically from authority or severity — footnoted deviations in matrix §2 document why. Changing a rule's gate class requires the same evidence discipline as changing its statement.

## 7. Review of the change

- Every diff is reviewed by a maintainer against this document.
- CAPire evidence is re-fetched live at review time (never taken on trust).
- Affected [operational validation scenarios](operational-validation.md) are identified BEFORE merge; the change is not accepted until they either still pass or are updated with a documented reason.
- If the change touches the [matrix](../development/rule-milestone-matrix.md), verify the rule row still contains 9 cells and the row count remains 134.

## 8. Amendment

Amendments preserve history:

- Rule IDs are **stable and never reused** (see §11).
- The rule's `Last verified` date is bumped.
- The change is summarized in [CHANGELOG.md](../CHANGELOG.md) with the rule ID and a one-line summary.
- If the amendment changes the *rule statement*, an inline note or matrix footnote records the change date and pilot/calibration reference (see the SRV-002 Detection guidance step 3 clarification and the SEC-013/ERR-005 matrix footnotes ³/⁴ as templates).

Silent rewrites of normative history are prohibited: a reader looking at git history + CHANGELOG must be able to reconstruct what the rule said at any prior point.

## 9. Deprecation

A rule that a future CAP version makes obsolete (mechanism removed, or SAP explicitly withdraws the guidance):

1. Change `Status` in the rule header from `Active` to `Deprecated`.
2. Record the SAP source establishing the deprecation.
3. Add an *Effective* date after which the rule no longer participates in gate evaluation.
4. Leave the rule body intact so historic reviews remain interpretable.

Deprecated rules are not removed from the catalog; they are visible historical record.

## 10. Retirement

If a rule is retired outright (not just superseded by a new SAP mechanism):

1. Move to `Status: Retired` with the retirement date and rationale.
2. Remove the rule from the [matrix](../development/rule-milestone-matrix.md) (footnoted).
3. Keep the file in `standards/rules/` — the ID is not reused (§11).

## 11. Rule IDs — stable and never reused

- Rule IDs are `CAP-<CAT>-<3-DIGIT-NUM>` with `<CAT>` from the 20 fixed categories.
- Once assigned, IDs **never** point to a different rule.
- A retired rule keeps its ID; a new rule gets the next unused number in that category.
- Reviewers cite rule IDs; the guarantee that a citation is stable underpins the audit trail.

## 12. Cross-references

- Cross-references are added at authoring time and re-checked during the [quarterly sweep](re-verification-cadence.md).
- Every rule's *Related rules* field lists rule IDs; when a related rule is deprecated/retired, update the cross-reference.
- The [audit tooling](#) (Part 24 of the v1.0 finalization report) validates cross-references at CI time; a broken cross-reference blocks merge.

## 13. Layer-3 (AI rules) governance

Layer-3 rules (`standards/ai/…`) follow the same lifecycle with two differences:

- Authority is typically **AI-REC** (Claude's operational rules), plus rare SAP-REC for AI rules that mirror SAP guidance.
- SAP evidence collection is not required for pure AI-workflow rules (there is no SAP source for how Claude should behave); the evidence base is instead the pilot/calibration record + [ai/README.md](../standards/ai/README.md)'s design.

## What this document is not

Not a change-review checklist for CI (that is script-level tooling). Not the authority taxonomy (see [authority-levels.md](authority-levels.md)). Not the milestone/gate model (see [rule-milestone-matrix.md](../development/rule-milestone-matrix.md)). This document explains *how* rules change; the referenced documents define *what* they are.
