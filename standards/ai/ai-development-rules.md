# AI-DEV — Development Rules for Claude Code

Authority: **ORG** (all rules). Runtime: Both. Status: Active.
These rules govern Claude Code when it **implements or modifies** a CAP application. The session flow that applies them is [development/development-model.md](../../development/development-model.md).

---

## AI-DEV-001 — Read applicable standards before writing code
**Severity: High.** Before implementing, Claude MUST load the Layer 2 rule categories applicable to the current milestone ([lifecycle](../../development/lifecycle.md)) and any project-local `CLAUDE.md`/ADRs. Work that begins without this context is invalid regardless of outcome.
*Detection:* the completion report lists the standards consulted.

## AI-DEV-002 — Inspect before proposing
**Severity: High.** Claude MUST inspect the actual repository (structure, `package.json`/`pom.xml`, CDS models, service definitions, handlers, configuration, tests) before proposing designs or changes. Proposals MUST reference observed facts (files, versions, patterns), not assumptions about "typical" CAP projects.
*Detection:* design proposals cite concrete file paths and observed versions.

## AI-DEV-003 — Restate requirements and surface ambiguity
**Severity: High.** Claude MUST restate its understanding of the requirement before implementing and MUST surface ambiguities, conflicts with the standard, or missing acceptance criteria as explicit questions or documented assumptions — never resolve them silently.
*Detection:* completion report contains a requirements summary and an assumptions list.

## AI-DEV-004 — CAP-native capability first
**Severity: High.** For every requirement, Claude MUST first identify whether CAP provides a native capability (generic CRUD handlers, CDS annotations for validation/authorization, drafts, localized data, temporal data, messaging, MTX, media handling, pagination, `cds.test`, …) and use it. Custom implementations of things CAP already does require a documented justification naming the CAP capability rejected and why.
*Detection:* custom code duplicating documented CAP capabilities without an ADR/justification is a violation.

## AI-DEV-005 — Preserve existing architecture
**Severity: High.** Changes MUST follow the architectural patterns already established in the project (module layout, naming, handler organization, error handling, test structure) unless the change's purpose is an approved refactoring. Local consistency beats global preference.
*Detection:* diffs match surrounding conventions; deviations are justified in the completion report.

## AI-DEV-006 — Implement incrementally
**Severity: Medium.** Work MUST proceed in small, coherent increments — model, then service, then logic, then tests, per the lifecycle — with each increment leaving the project buildable and its tests passing, rather than large big-bang changes.
*Detection:* commit/change structure shows coherent increments; intermediate build/test evidence in the report.

## AI-DEV-007 — Tests accompany code
**Severity: High.** Every behavior added or changed MUST come with new or updated tests in the same increment (see [AI-TEST rules](ai-testing-rules.md)). "Tests later" is a violation, not a plan.
*Detection:* diff contains test changes corresponding to production changes.

## AI-DEV-008 — Self-validate before declaring done
**Severity: High.** Before reporting completion, Claude MUST (a) run the build and full relevant test suite, (b) check its changes against the applicable Layer 2 rules, and (c) record the result per rule (met / not met / not applicable) in the completion report. Compliance claims without evidence are prohibited (see AI-DEV-016).
*Detection:* completion report contains the self-validation table and test output summary.

## AI-DEV-009 — Distinguish fact from recommendation in all outputs
**Severity: High.** In designs, code comments, and reports, Claude MUST label SAP-documented facts (with references), general practice, and its own recommendations per [authority levels](../../docs/authority-levels.md), and MUST NOT attribute to SAP anything it cannot cite.
*Detection:* spot-check claims labelled SAP-REQ/SAP-REC against their references.

---

## Guardrails — prohibited without explicit human approval

The following rules are all **Severity: Critical**. Violating any of them invalidates the work product even if it functions.

## AI-DEV-010 — No unapproved architectural change
Claude MUST NOT introduce major architectural changes — new services, changed module boundaries, changed persistence strategy, changed deployment target, new runtime — without explicit approval of a written proposal.

## AI-DEV-011 — No unnecessary frameworks or abstractions
Claude MUST NOT introduce new frameworks, libraries, or bespoke abstraction layers (custom ORM wrappers, custom validation frameworks, custom auth layers, generic "base handler" hierarchies) where CAP-native or already-present means suffice.

## AI-DEV-012 — No bypassing CAP abstractions without justification
Claude MUST NOT bypass CAP abstractions (e.g., raw SQL instead of CQL/CQN, hand-rolled HTTP handling instead of protocol adapters, direct HTTP calls instead of remote-service consumption, manual transaction handling where managed transactions apply) unless a documented, approved justification exists.

## AI-DEV-013 — No weakening of security
Claude MUST NOT remove or loosen authorization annotations, disable authentication (including leaving mocked auth reachable in production profiles), widen CORS, expose internal services/entities, log secrets, or downgrade validation — under any circumstances, including "to make it work". Security-relevant changes require explicit approval and a security note in the completion report.

## AI-DEV-014 — No deleting or gutting tests to achieve green
Claude MUST NOT delete, skip, weaken, or trivially satisfy failing tests to make a suite pass. A failing test is either a defect to fix in production code or a test to fix with justification recorded — decided visibly, never silently.

## AI-DEV-015 — No silent deviation from the standard
If Claude cannot or should not comply with an applicable rule, it MUST say so explicitly (rule ID, reason, proposed exception) in the session and in the completion report. Silently ignoring a standard is a Critical violation even when the deviation itself would be defensible.

## AI-DEV-016 — No compliance claims without evidence
Claude MUST NOT state that code complies with a rule, that tests pass, or that a build succeeds unless it has actually verified it in the session and can show the evidence (command output, file references). Predicted or assumed results MUST be labelled as unverified.

## AI-DEV-017 — Uncertainty is stated, not smoothed over
**Severity: High.** Where CAP documentation is ambiguous, version-dependent, or silent, Claude MUST say so, record the gap ([research-gaps.md](../../references/research-gaps.md) pattern), choose the most defensible option, and label it `AI-REC` — rather than presenting a guess as settled guidance.
