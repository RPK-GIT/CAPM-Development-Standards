# Development Model — Claude Code as Implementer

How a Claude Code development session on a CAP application proceeds under this standard. The binding rules are the [AI-DEV](../standards/ai/ai-development-rules.md), [AI-TEST](../standards/ai/ai-testing-rules.md), and [AI-DOC](../standards/ai/ai-documentation-rules.md) families; this document is the operational sequence that applies them. Claude Code is *one possible* implementation tool — human developers follow the same sequence with the same gates.

## Session sequence

### 1. Read applicable standards *(AI-DEV-001)*
Identify the current [milestone](lifecycle.md) and load its applicable Layer 2 categories, the Layer 3 rules, the project's `CLAUDE.md`, and existing ADRs/exception records.

### 2. Inspect the existing implementation *(AI-DEV-002)*
Establish the project profile from the repository itself: runtime and versions (`package.json`/lockfile or `pom.xml`), project layout, CDS models, services, handlers, configuration profiles, deployment artifacts, test setup, tenancy model. Record the profile — it scopes which rules apply.

### 3. Understand requirements *(AI-DEV-003)*
Restate the requirement, its acceptance criteria, and its boundaries. List ambiguities and either ask (interactive session) or record explicit assumptions (autonomous session).

### 4. Identify applicable CAP capabilities *(AI-DEV-004)*
For each part of the requirement, name the CAP-native capability that covers it (generic handlers, annotations, drafts, messaging, remote services, MTX, …) — with the SAP doc reference — before considering custom code. Custom code is the residual, not the default.

### 5. Propose / confirm architecture *(AI-DEV-005, AI-DEV-010)*
If the work fits existing architecture: state that and proceed. If it requires architectural change: write the proposal (options, impact, affected rules) and **stop for approval**.

### 6. Implement incrementally *(AI-DEV-006, AI-DEV-011…013)*
Small increments in lifecycle order (model → service → logic → integration), each leaving the project buildable and green. Respect the guardrails at every step.

### 7. Add / update tests *(AI-DEV-007, AI-TEST-001…005)*
Tests accompany each increment: behavior through service interfaces, unhappy paths included, CAP-native test tooling, deterministic.

### 8. Validate against applicable rules *(AI-DEV-008)*
Run build and tests (AI-TEST-006). Walk the applicable Layer 2 rules and record met / not met / N/A per rule with evidence.

### 9. Perform self-review
Re-read the full diff as a skeptical reviewer: consistency with surrounding code, security impact, missed unhappy paths, leftover debug artifacts, documentation drift (AI-DOC-005). Fix what surfaces.

### 10. Produce a completion report *(AI-DOC-003)*
Use [templates/completion-report-template.md](../templates/completion-report-template.md): what changed, requirements coverage, self-validation table, test evidence, decisions and assumptions, deviations/exceptions, open items. The report is input to — never a substitute for — the independent milestone review (AI-REVIEW-009).

## Hard boundaries — never without explicit approval

Per AI-DEV-010 … AI-DEV-016 (all Critical), Claude must not:

1. Introduce major architectural changes (services, boundaries, persistence, runtime, deployment target).
2. Invent frameworks or abstraction layers where CAP-native or existing means suffice.
3. Bypass CAP abstractions (raw SQL over CQL, hand-rolled HTTP over adapters/remote services, manual transactions where managed ones apply) without documented justification.
4. Weaken security in any form — including leaving dev/mock auth reachable from production profiles.
5. Remove, skip, or gut tests to make a suite pass.
6. Silently ignore an applicable standard — deviations are declared, not buried.
7. Claim compliance, passing tests, or successful builds without verified evidence.

If any of these appears necessary, the session stops and escalates with a written proposal.

## Escalation and uncertainty

- Conflict between requirement and standard → surface it; the standard wins until an exception is approved (AI-DEV-015, AI-DOC-002).
- Conflict between two rules → resolve via the [Layer 1 principles](../standards/principles/engineering-principles.md); record the resolution.
- SAP guidance ambiguous or absent → AI-DEV-017: state the gap, choose the most defensible option, label it `AI-REC`, and add it to the gap register pattern.
