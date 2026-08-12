# Operational Validation Scenarios

Claude-executable test scenarios for the `/capm-develop` and `/capm-review-milestone` workflows. Consistent with the repository's documentation-only constraint, these are **specified scenarios with defined expected outcomes** — executed by Claude Code against fixtures (see the [worked example](../examples/worked-example-m3/README.md)), not by committed scripts. Run the relevant scenarios after any change to the commands, the [profile spec](../development/project-profile.md), the [matrix](../development/rule-milestone-matrix.md), or the [CAPire protocol](../reviews/capire-verification.md).

Each scenario: **Given** (input/fixture) → **Expect** (required outcome). A scenario fails if the actual behavior deviates in substance.

## P — Profile validation

| # | Given | Expect |
|---|---|---|
| P1 | Valid Node.js profile (worked-example profile) | Validation passes; runtime filter = nodejs |
| P2 | Same profile with `runtime: java`, `version: "5"` | Validation passes; Node-only rules excluded, Java-only included |
| P3 | Profile with `cap.runtime` missing | **NOT-ASSESSABLE**, naming `cap.runtime`; no guessed value; repository-derived value may only be *proposed* |
| P4 | `multitenant: true` + `database.production: postgres` | ✗ invalid combination → NOT-ASSESSABLE until corrected |
| P5 | `pdm: true` + `personal_data: false` | ✗ invalid combination |
| P6 | `milestone: M3` + `identity.service: undecided` | ✗ (required from M1) |
| P7 | Profile says `eventing: false` but repo contains messaging config | CONTRADICTORY profile finding; review stops for profile correction |
| P8 | Profile contains a `clientsecret`-like value | CAP-SEC-017 finding; profile-security constraint cited |

## F — Rule filtering

| # | Given | Expect |
|---|---|---|
| F1 | Node profile, M4 review | CAP-LOGIC-004/-005, CAP-TXN-004, CAP-DB-010 (Java-only) absent from the evaluated set |
| F2 | Java profile, M4 review | CAP-LOGIC-003, CAP-TXN-002/-003/-006, CAP-DB-005/-006/-009 (Node-only) absent |
| F3 | `eventing: false`, M5 review | CAP-EVT-* NOT APPLICABLE (not findings); if also `remote_services: false` and `extensibility: false` → milestone NOT APPLICABLE |
| F4 | `multitenant: true`, M6 review | CAP-MT-003/-004 included; MT FINAL-GATE items appear at M8/M9 |
| F5 | Version-sensitive rule (e.g., CAP-SRV-005) in any review | Register (`docs/version-management.md`) consulted; its verified date checked before use; no hard-coded versions |
| F6 | M3 review, worked-example profile | Exactly the 10-rule set of the worked example (15 − 5 filtered) |

## G — Gate behavior

| # | Given | Expect |
|---|---|---|
| G1 | Critical rule FAIL (CAP-SEC-001, no exception) | Milestone **FAIL**; finding prominent in the Critical subsection; never silently downgraded |
| G2 | HARD (non-Critical) FAIL, e.g. CAP-SEC-011 | Milestone FAIL unless a valid exception record exists |
| G3 | SOFT FAIL only (e.g., CAP-SRV-006) | Reported with impact + remediation; milestone may PASS only with explicit recorded justification |
| G4 | ADVISORY finding only | Recommendation in the report; never blocks |
| G5 | HARD FAIL + valid AI-DOC-002 exception (in scope, unexpired, approver named) | **PASS WITH EXCEPTIONS**; exception referenced in §9 |
| G6 | HARD FAIL + expired/out-of-scope exception | FAIL; exception rejected with reason |
| G7 | ORG-PENDING rule FAIL (e.g., CAP-OPS-003 at M9) | Reported as ORG-PENDING policy finding; blocking attributed to this standard's governance, never to SAP |

## E — Evidence handling

| # | Given | Expect |
|---|---|---|
| E1 | DIRECT evidence (annotation present at file:line) | PASS with file:line citation |
| E2 | Expected artifact absent (no `@requires` anywhere) | Verified absence recorded as evidence; FAIL — not NOT-ASSESSABLE (absence *is* determinable) |
| E3 | Evidence CONTRADICTORY (config says X, code does Y) | FAIL with both citations |
| E4 | Evidence genuinely unavailable (needs runtime/platform export) | NOT ASSESSABLE naming the exact evidence needed; never PASS |
| E5 | Developer assertion only ("we checked it") | Not accepted as evidence unless the rule explicitly accepts a documented decision (e.g., CAP-SRV-008's recorded last-write-wins) |

## C — CAPire verification

| # | Given | Expect |
|---|---|---|
| C1 | All sources CURRENT | §10 table complete; verdicts stand |
| C2 | A source page materially changed vs the rule's claim | Rule NOT ASSESSABLE (if materially dependent); governance flag raised; rule NOT rewritten during review |
| C3 | Source REMOVED (404, no successor) | As C2 — unless the rule anchors a live alternative source (CAP-ARCH-002 pattern) |
| C4 | Version-sensitive rule + register verified-date stale | Level 3 requires re-verification before the rule is assessed; report says so |
| C5 | Fetch UNAVAILABLE (offline), register within cadence | Verdict stands; report states "live check unavailable; relying on register verification of <date>" |
| C6 | Critical/security rule in scope | Level 2 explicit fetch performed (not just URL liveness) |

## R — Report content

| # | Given | Expect |
|---|---|---|
| R1 | Any completed review | Report contains: profile summary, per-rule verdicts + evidence, HARD/SOFT/ADVISORY findings separated, Critical subsection, §10 CAPire table with source statuses, exceptions, remediation plan, gate result, next-milestone readiness |
| R2 | Any completed review | Scope claim is precise ("applicable Mx standards evaluated in this review") — no blanket best-practice claims |
| R3 | Re-review after remediation | Scope = previously FAILED/NOT-ASSESSABLE rules + touched files; resolution verified per rule detection, not inferred from file changes |
| R4 | Review of any kind | Zero application-code modifications occurred (read-only check) |

## D — Development mode

| # | Given | Expect |
|---|---|---|
| D1 | `/capm-develop` with valid profile at M4 | Loaded rule set = M4 primaries filtered by profile — not all 134 |
| D2 | Task requiring a service-boundary change | STOP + architectural-deviation report (AI-DEV-010); no silent change |
| D3 | Failing test blocking a task | Test fixed or defect fixed — never deleted/weakened (AI-DEV-014) |
| D4 | Completion | Report per template with self-validation table and evidence locations for review handoff |

**Reference execution:** the [worked example](../examples/worked-example-m3/review-report.md) instantiates P1, F5, F6, G1, G3, E1, E2, E4/E5 (SRV-008), C1, C6, R1, R2.
