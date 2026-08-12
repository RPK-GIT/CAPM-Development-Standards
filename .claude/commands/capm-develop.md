# /capm-develop — Develop against the CAPM Engineering Standard

You are implementing work in a CAP project that adopts the CAPM Development Standards. Task (optional, after the command): $ARGUMENTS

Follow the repository's existing models — this command *binds* them, it does not replace them: the ten-step [development model](../../development/development-model.md), the [Layer 3 AI rules](../../standards/ai/README.md) (AI-DEV/AI-TEST/AI-DOC in full), the [rule-milestone matrix](../../development/rule-milestone-matrix.md), and the current milestone's [checklist](../../development/milestones/m0-requirements.md). When this command runs inside an adopting project, resolve these paths inside the vendored/referenced standards repo named in `capm-profile.yaml → standard.version`.

## Procedure (= development-model steps 1–10, operationalized)

1. **Read applicable standards** *(step 1)* — do NOT load all 134 rules. Load: the current milestone's checklist, plus the matrix rows selected in step 5 below.
2. **Load and validate the profile** — read `capm-profile.yaml`; validate per [project-profile.md](../../development/project-profile.md). Missing/invalid → **STOP, return NOT-ASSESSABLE** listing the missing fields; propose repository-derived values but never write them yourself without user acceptance. No profile at all → offer to create one from [the template](../../templates/capm-profile-template.yaml) via M0.
3. **Inspect the existing implementation** *(step 2)* — per AI-DEV-002: structure, manifests, models, services, handlers, config, tests. Record contradictions between profile and repository as findings before proceeding.
4. **Understand the requirement** *(step 3)* — restate it; surface ambiguity (AI-DEV-003).
5. **Select the applicable rule set** — from the matrix: the current milestone's PRIMARY + FINAL-GATE rows, plus SUPPORTING rows whose subjects the task touches; filter by `cap.runtime` and every profile capability flag. Extract from the selected rows: **the HARD-GATE constraints** (design so they cannot fail) and **the evidence each rule expects** (produce it as you work). Name CAP-native capabilities per requirement part (AI-DEV-004) before writing custom code.
6. **Propose/confirm architecture fit** *(step 5)* — if the work fits existing architecture, say so and proceed. If it requires any protected change — service boundaries, persistence strategy, new infrastructure/frameworks, runtime, authentication/authorization weakening, test removal, tenancy architecture (AI-DEV-010…-016) — **STOP and report the architectural deviation**; proceed only with explicit approval recorded per AI-DOC-002/CAP-ARCH-007.
7. **Implement incrementally** *(step 6)* — small increments, buildable and green after each (AI-DEV-006); runtime- and version-correct per the selected rows and the [version register](../../docs/version-management.md).
8. **Tests accompany code** *(step 7)* — per AI-TEST-001…-007; never weaken/delete tests to pass (AI-DEV-014).
9. **Self-validate** *(step 8)* — run build + tests (actually run them, AI-TEST-006); walk the selected rule rows, record met / not met / N/A with evidence per rule. Then self-review the full diff *(step 9)*.
10. **Report** *(step 10)* — produce the [completion report](../../templates/completion-report-template.md) including the self-validation table, the evidence produced (for the review workflow to consume), assumptions, deviations/exceptions. Follow git safety: inspect status/diff, no secrets/env files, no history rewrites; commit only when the user asked.

## Development→review handoff

Persist evidence where the review will look for it (in the adopting project): `capm-profile.yaml` (current), `docs/adr/` (decisions, CAP-ARCH-007), `docs/capm/` (completion reports, milestone checklist instances, exception records per AI-DOC-002), tests + CI output. Traceability chain: requirement → implementation → test → evidence → rule → review.

## Hard boundaries (from AI-DEV-010…-016 — binding, not advisory)

Never without explicit approval: architectural changes, CAP-duplicating frameworks, bypassed CAP abstractions, weakened security, gutted tests, silently ignored standards, unevidenced compliance claims. When blocked on one of these: stop, report, escalate — do not improvise around the standard.
