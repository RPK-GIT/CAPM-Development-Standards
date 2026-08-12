# Operational Validation — Execution Record (2026-08-12)

Execution of all 36 scenarios from [operational-validation.md](operational-validation.md) against the [Node.js fixture](../examples/worked-example-m3/README.md) and the [Java fixture](../examples/worked-example-m3-java/README.md). **Execution modes** (stated per scenario, honestly): **FIXTURE** = exercised against a committed fixture artifact; **TRANSIENT** = exercised against a constructed input shown below; **WALKTHROUGH** = procedure executed step-by-step where no real trigger exists today (e.g., no genuinely changed SAP source), with the expected branch verified against the written protocol. Live CAPire fetches performed 2026-08-12 are listed in §CAPire.

**Result: 36/36 PASS** — with one genuine defect *found and fixed by this execution* (calibration item 1: a false negative in the Node example report), which is the point of validation.

## P — Profile (8/8 PASS)

| # | Mode | Expected | Actual | Result |
|---|---|---|---|---|
| P1 | FIXTURE | Node profile validates; runtime filter = nodejs | All required fields present, values allowed, no ✗ combos; plausibility vs `package.json` (`@sap/cds ^10`) consistent | **PASS** |
| P2 | FIXTURE | Java profile validates; Java-only rules included downstream | Java profile validates; `pom.xml` (`cds-services-bom 5.0.0`, `java.version 21`) consistent; Java-only inclusion proven in §F2 | **PASS** |
| P3 | TRANSIENT | Missing `cap.runtime` → NOT-ASSESSABLE, named, not guessed | Node profile with `cap.runtime` deleted → validation step 2 returns NOT-ASSESSABLE listing `cap.runtime`; a *proposal* (`nodejs`, from package.json) is offered but not written | **PASS** |
| P4 | TRANSIENT | `multitenant: true` + `postgres` → ✗ | Constructed combo hits invalid-combination row 1 → NOT-ASSESSABLE until corrected | **PASS** |
| P5 | TRANSIENT | `pdm: true` + `personal_data: false` → ✗ | Hits row 4 → NOT-ASSESSABLE | **PASS** |
| P6 | TRANSIENT | M3 + `identity.service: undecided` → ✗ | Hits row 3 (required from M1) → NOT-ASSESSABLE | **PASS** |
| P7 | TRANSIENT | Profile `eventing: false` + repo messaging config → CONTRADICTORY, stop | Node fixture variant with `cds.requires.messaging` added to package.json → validation step 4 → CONTRADICTORY profile finding; review stops for correction | **PASS** |
| P8 | TRANSIENT | Secret-like value in profile → CAP-SEC-017 finding | Profile with `clientsecret: "aBcD1234…"` → flagged citing the profile-security constraint + CAP-SEC-017 | **PASS** |

## F — Filtering (6/6 PASS)

| # | Mode | Expected | Actual | Result |
|---|---|---|---|---|
| F1 | FIXTURE | Node M4 set excludes Java-only rules | Node fixture M4 selection = **20 rules**: 13 Both (LOGIC-001/-002, DB-004/(-008 out: no locking), TXN-001/-005, SEC-013/-016, ERR-001/-003/-005/-006, LOG-001, PERF-007) + 7 Node-only (LOGIC-003, TXN-002/-003/-006, DB-005/-009, ERR-002); DB-006 (LOCALIZED), PERF-005 (MASS-DATA), ERR-004 (no error hooks in fixture), MT-006 (MT) filtered out. **CAP-LOGIC-004/-005, CAP-TXN-004, CAP-DB-010 absent** ✓ | **PASS** |
| F2 | FIXTURE | Java M4 set excludes Node-only rules | Java fixture M4 selection = **17 rules**: the same 13 Both + **4 Java-only (LOGIC-004/-005, TXN-004, DB-010)**; all 10 Node-only rules (LOGIC-003, DB-005/-006/-009, TXN-002/-003/-006, ERR-002/-004, PERF-005) absent ✓ | **PASS** |
| F3 | FIXTURE | `eventing/remote/extensibility: false` → M5 NOT APPLICABLE | Both fixtures: all M5 capability groups flagged false → milestone NOT APPLICABLE with the profile as evidence; no CAP-EVT/INT/EXT findings anywhere | **PASS** |
| F4 | TRANSIENT | `multitenant: true` → MT rules included | Java profile variant (multitenant true, sidecar true, hana) → M6 set gains MT-003/-004; M8 gains MT-005 + MT-002 FG | **PASS** |
| F5 | FIXTURE | Version-sensitive rule consults the register | CAP-SRV-005 evaluation in both reports first confirmed [version-management.md](version-management.md) verified-date (2026-08-12, current); no hard-coded versions used | **PASS** |
| F6 | FIXTURE | Node M3 = exactly the 10-rule set | 15 primaries − SRV-007/SRV-009/SEC-010/SEC-018/PERF-001 = the 10 evaluated (script-verified against the matrix in Prompt 2; re-confirmed) | **PASS** |

## G — Gates (7/7 PASS)

| # | Mode | Expected | Actual | Result |
|---|---|---|---|---|
| G1 | FIXTURE | Critical FAIL → milestone FAIL, prominent | Node: CAP-SEC-001 (Critical, HARD) FAIL → gate recommendation FAIL; finding is §3's first block with full evidence chain | **PASS** |
| G2 | FIXTURE | HARD (non-Critical) FAIL → FAIL | Java: CAP-SEC-011 (High, HARD) FAIL, no exception → milestone FAIL | **PASS** |
| G3 | FIXTURE | SOFT FAIL reported, no auto-FAIL | CAP-SRV-006 FAIL in both fixtures reported with impact+remediation; milestone FAILs came only from the HARD findings — in the re-reviews, after HARD resolution, no SOFT-caused FAIL occurred | **PASS** |
| G4 | TRANSIENT | ADVISORY reported only | M2 mini-check on a constructed model (`entity book { key book_id : UUID; }`) → CAP-CDS-001 (ADV) reported as recommendation; explicitly non-blocking | **PASS** |
| G5 | TRANSIENT | Valid exception → PASS WITH EXCEPTIONS | Node fixture + constructed AI-DOC-002 record for CAP-SEC-001 (approver: fictional CISO, scope: OrdersService, expiry 2026-12, compensating control: network policy) → HARD gate covered → PASS WITH EXCEPTIONS, exception cited in §9 | **PASS** |
| G6 | TRANSIENT | Expired exception rejected | Same record with expiry 2026-06 → rejected ("expired 2026-06-30"), FAIL stands | **PASS** |
| G7 | WALKTHROUGH | ORG-PENDING finding attributed correctly | M9 walkthrough with missing readiness assessment → CAP-OPS-003 FAIL reported as *ORG-PENDING policy finding; blocking authority derives from this standard's governance, not SAP* — wording per matrix §1.6 | **PASS** |

## E — Evidence (5/5 PASS)

| # | Mode | Expected | Actual | Result |
|---|---|---|---|---|
| E1 | FIXTURE | DIRECT → PASS with file:line | Java CAP-SEC-001 PASS: `approval-service.cds` L3 | **PASS** |
| E2 | FIXTURE | Verified absence → FAIL (not N-A) | Node CAP-SEC-001: absence of any authorization annotation across `srv/**` recorded as evidence → FAIL | **PASS** |
| E3 | TRANSIENT | CONTRADICTORY → FAIL with both citations | Constructed: profile `drafts: false` + `@odata.draft.enabled` in a service → contradiction cited from both sides (profile line + cds line) | **PASS** |
| E4 | FIXTURE | Missing evidence → NOT ASSESSABLE naming the need | Node CAP-SRV-008 initial review: no annotation, no decision record → NOT ASSESSABLE with the three acceptable evidence forms named | **PASS** |
| E5 | FIXTURE | Assertion ≠ evidence unless rule permits documented decision | CAP-SRV-008 demonstrates the boundary: a *recorded* last-write-wins decision would count (rule text); a bare claim would not — initial review found neither | **PASS** |

## C — CAPire (6/6 PASS) — live fetch record of 2026-08-12

Live fetches this execution: `guides/security/authorization` → **CURRENT** (no-default-access + Java composition gap + Node target-only evaluation re-confirmed verbatim); `java/cqn-services/application-services` → **CURRENT** (V4+V2 default re-confirmed); `node.js/cds-serve` → **CURRENT** (`@protocol:'none'`, absolute-path constraint re-confirmed); `about/bad-practices` → **HTTP 404** (REMOVED re-confirmed).

| # | Mode | Expected | Actual | Result |
|---|---|---|---|---|
| C1 | FIXTURE+LIVE | CURRENT sources; verdicts stand | Three live fetches CURRENT (above); remaining evaluated-rule URLs covered by the register (verified 2026-08-12, within cadence) — reports state which | **PASS** |
| C2 | WALKTHROUGH | Changed source → N-A + governance flag, no rewrite | No genuinely changed source exists today; protocol branch walked: authorization page hypothetically dropping the no-default-access statement → CAP-SEC-001 NOT ASSESSABLE + governance flag + no rule edit (protocol §Change handling) | **PASS** |
| C3 | **LIVE** | Removed source → N-A unless anchored alternative | **Real case executed:** `about/bad-practices` fetched → 404. Dependent rule CAP-ARCH-002 *anchors a live alternative* (get-started/concepts, per its rule text) → assessable via anchor. A rule without an anchor would be NOT ASSESSABLE — both branches demonstrated with a real REMOVED source | **PASS** |
| C4 | WALKTHROUGH | Stale register blocks version-sensitive assessment | Constructed: register verified-date set to 2026-01 → Level 3 requires register re-verification before CAP-SRV-005/VER-scoped assessment; report must state the block | **PASS** |
| C5 | WALKTHROUGH | Offline fallback honest | Network live today; fallback wording verified against protocol: "live check unavailable; relying on register verification of <date>" only within cadence, else STALE-treatment | **PASS** |
| C6 | FIXTURE+LIVE | Level 2 explicit fetch for Critical/security | CAP-SEC-001/CAP-SEC-011 verified by explicit statement-level fetch (not URL liveness) — quotes re-confirmed | **PASS** |

## R — Report (4/4 PASS)

| # | Mode | Actual | Result |
|---|---|---|---|
| R1 | FIXTURE | Both reports checked section-by-section against the template: profile summary, per-rule verdicts+evidence, HARD/SOFT separated, Critical prominent, §10 CAPire table with statuses, exceptions, remediation, gate result, next-milestone readiness — all present | **PASS** |
| R2 | FIXTURE | Both reports end with the precise scope statement; no blanket best-practice claims anywhere | **PASS** |
| R3 | FIXTURE | Both [re-reviews](../examples/worked-example-m3/re-review.md) scope = prior FAIL/N-A + touched files; every verdict from a **re-run of detection** (the Node re-review's CAP-SEC-012 re-check and the Java CAP-SEC-012 orphan-check demonstrate non-inference explicitly) | **PASS** |
| R4 | FIXTURE | Reviews modified no fixture content; remediation happened as a separate, explicit step between review and re-review | **PASS** |

## D — Develop mode (4/4 PASS)

Exercise: small controlled change on the Node fixture — *"add a `cancel` action to OrdersService"* — walked through `/capm-develop` (no large feature, per scope):

| # | Mode | Actual | Result |
|---|---|---|---|
| D1 | FIXTURE | Profile loaded (M3); selected set = M3 primaries filtered (10) + touched-subject SUPPORTING (CAP-ERR-001 error contract) — **not 134**; HARD constraints extracted up front (SEC-001: the new action inherits `OrdersService`'s restriction — which the fixture initially lacks → the develop flow itself surfaces the SEC-001 problem *before* implementing) | **PASS** |
| D2 | TRANSIENT | Variant task *"store orders in a separate PostgreSQL for reporting"* → step 6 detects a persistence-strategy change → **STOP + architectural deviation report** (AI-DEV-010); no implementation performed | **PASS** |
| D3 | WALKTHROUGH | Failing-test policy verified against AI-DEV-014 binding in the command: fix defect or fix test with justification — deletion/weakening path does not exist in the procedure | **PASS** |
| D4 | FIXTURE | Completion-report fields produced for the exercise: requirement restatement, CAP-native mapping (bound action + `@On`-analog handler), self-validation table over the selected rules, evidence locations (`srv/orders-service.cds`, test file) for review handoff | **PASS** |

## M3 review comparison (Part 5) — from actual fixture execution

| Dimension | Node.js (granary-orders) | Java (millhouse-approvals) |
|---|---|---|
| Runtime | Node.js, `@sap/cds` 10 | Java, CAP Java 5 |
| M3 primary set | 15 | 15 |
| Filtered out (capability) | 5 | 6 |
| **Applicable rules evaluated** | **10** | **9** |
| Node-only rules in set | 0 *(M3 has no runtime-scoped primaries)* | 0 ✓ excluded by construction |
| Java-only rules in set | 0 ✓ | 0 *(none exist at M3; see M4 demo)* |
| Both-runtime rules | 10 | 9 |
| Runtime-specific checks inside Both-rules | SRV-006 via `cds-serve` semantics | SRV-006 via the **Java V4+V2 default**; SRV-004's Java `@On` duty |
| HARD gates in scope | 2 (SEC-001, SEC-011) | 2 (SEC-001, SEC-011) |
| Result | FAIL (1 HARD, 2 SOFT, 1 N-A) → re-review **PASS** | FAIL (1 HARD, 1 SOFT) → re-review **PASS** |
| **M4 selection demo (runtime proof)** | **20 rules** (7 Node-only in, 0 Java-only) | **17 rules** (4 Java-only in, 0 Node-only) |

## Negative tests (Part 9) — all covered above
Invalid profile → P3 · wrong runtime/contradiction → P7 · missing capability declaration → P3/P4-P6 pattern (undeclared capability = missing required field → NOT-ASSESSABLE, never guessed) · missing evidence → E4 · HARD FAIL → G1/G2 · SOFT → G3 · ADVISORY → G4 · removed source → C3 (real) · stale register → C4.

## Calibration findings (Part 11)

| # | Type | Rule | Problem & evidence | Recommendation | When |
|---|---|---|---|---|---|
| 1 | **False negative (found & fixed)** | CAP-SRV-001 | The original Node example report passed SRV-001 while `OrdersService` re-exposed `Orders` 1:1 incl. `internalMargin` — detection steps 3–4 clearly fire. The re-review's own remediation (excluding the field) exposed the inconsistency. **Report corrected** (fixture and rule were right; the report execution missed a service) | Reinforced: detection step 1 iterates **per service** — reviewers must complete the iteration, not stop at the first compliant service. No rule change needed | Fixed now |
| 2 | Overlap/noise risk | CAP-SRV-005 / CAP-SRV-006 | In Java, the *same fact* (undecided default V2 exposure) nearly double-reports: SRV-006 owns the undecided-default finding; SRV-005 would add a justification demand for the same endpoint | Convention adopted in the Java example (SRV-005 "PASS*" with subsumption note): **SRV-006 owns undecided defaults; SRV-005 binds decided V2 exposure.** Consider one clarifying sentence in both rules | Phase 4 |
| 3 | Ambiguity | Matrix SUPPORTING selection | "SUPPORTING rows whose subjects the milestone's changes touch" is reviewer judgment — two reviewers could scope differently | Define a per-category touch-map heuristic (file globs per category) in Phase 4 after pilot data | Phase 4 |
| 4 | Evidence realism | CAP-SEC-009 | Role-collection assembly often lives only in BTP cockpit → recurring partial NOT-ASSESSABLE | Already designed (rule names the cockpit-export evidence); pilot should confirm the export ask is practical | Phase 4 observation |
| 5 | Noise watch | CAP-CDS-001 | Naming ADV could chatter on legacy/external-shaped models | Existing carve-outs (imported models) look sufficient; verify on pilot | Phase 4 observation |

No false positives observed; no rule wording prevented objective evaluation (item 3 is a matrix/workflow refinement, not a rule defect). **No standard (Layer 2) changes were made** — Part 12 separation held: one fixture-report correction (item 1), zero workflow-spec changes, two Phase 4 research items.

## Phase 3 exit criteria (Part 13)

| Criterion | Status |
|---|---|
| 134/134 mapped, no orphans | ✅ (Prompt 1 validation, re-confirmed) |
| Node.js behavior validated | ✅ (fixture: full review, re-review, develop exercise) |
| Java behavior validated | ✅ (fixture: full review, re-review; M4 runtime-selection demo) |
| Milestone / capability / runtime / version filtering | ✅ (F1–F6) |
| Evidence classes | ✅ (E1–E5) |
| Hard / Soft / Advisory gates | ✅ (G1–G7) |
| CAPire: relevant sources, stale/removed, version-sensitive | ✅ (C1–C6; C3 executed against a *real* removed source) |
| Reports traceable, accurate, actionable | ✅ (R1–R4; calibration item 1 demonstrates the traceability catching its own gap) |
| Remediation + re-review | ✅ (both fixtures, FAIL → remediate → detection-re-run → PASS) |
| `/capm-develop` per standards | ✅ (D1–D4 incl. STOP behavior) |

**All Phase 3 exit criteria satisfied.** Caveat, stated plainly: validation ran against **fixtures**, not production projects — the standard is *validated*, not *production-proven*. That distinction is exactly what the Phase 4 pilot exists to close.
