# CAPM Standard Review Report — Pilot 1 · Round 2 — scs — M3

> **Phase 4 pilot review against the applicable CAPM Development Standards.** Not a certification; not a production-proven claim. *(Report per [review-report-template](../../templates/review-report-template.md); command: `/capm-review-milestone re-review M3`.)*

| Field | Value |
|---|---|
| **Project** | scs (Supply Coverage Simplified) — internal, deployed CAP application |
| **Reviewed revision** | `c1f51ac` + current working tree (2026-08-12; working-tree differences vs Round 1 = UI build output, one added remote-service destination `ZSB_MRPCONTROLLER_Private`, whitespace/formatting cleanups in `db/interactions.cds`, ~125 lines removed from `srv/interaction_srv.js` — none of which affect Round 1 findings; see §16) |
| **Review date** | 2026-08-12 |
| **Reviewer** | Claude Code (Round 2) |
| **Standard version** | `4fbaac7` (Round 1 calibration applied — matrix, SRV-002/SEC-014, profile spec, review command, report template) |
| **Scope** | Milestone gate review — M3 (Services/API), re-review of [Round 1](../pilot-1-scs-m3/review-report.md) |
| **Review type** | re-review of [Round 1 report](../pilot-1-scs-m3/review-report.md) |
| **Review mode** | RETROSPECTIVE — deployed application (`project.milestone: LIVE`) — **findings represent current operational risk** |
| **Review duration** | ≈ 50 min wall clock incl. live CAPire re-verification |

## 1. Project profile

From [`capm-profile.yaml`](../../../SupplyCoverageSimplified/capm-profile.yaml) at the project root (not copied here — see §23 for the anonymization posture). Profile is the same as Round 1 with the **calibration migration applied**: 4 workload flags moved from `false` to `unknown` (`mass_data`, `concurrent_paging`, `hana_large_volume`, `major_upgrade_in_scope`); `concurrent_edit: true # source: repo-evidence (global Variants IsGlobal + PurchaseReqnItemText custom UPDATE — shared writable outside drafts)`. **Owner has not provided input on the `unknown` flags between rounds** — they stay `unknown`, and CAP-PERF-001 correctly stays NOT ASSESSABLE (the workflow's calibration-designed behavior; A-1 / WCP-3 in action).

Profile validates OK (no ✗; no CONTRADICTORY plausibility findings — the `concurrent_edit: true` owner-declared/repo-evidence flag is corroborated by the current handlers — global Variants updatable, `PurchaseReqnItemText` custom UPDATE at L3849, both under `concurrent_edit: true`). No secrets present.

## 2. Verdict summary

| Set | PASS | FAIL | NOT APPLICABLE | NOT ASSESSABLE |
|---|---|---|---|---|
| PRIMARY (15) | 4 | 8 | 2 | 1 |
| SUPPORTING (5) | 2 | 3 | 0 | 0 |
| **Total (20 selected)** | **6** | **11** | **2** | **1** |

Gate class distribution unchanged: 3 HARD / 17 SOFT / 0 ADVISORY. Findings: **1 Critical HARD FAIL, 10 SOFT FAILs** (per-rule); consolidated into **7 root findings** (§15). Root-finding count is the executive-summary defect count; per-rule counts above are per-rule and stay per-rule (report template §3).

**Gate recommendation: FAIL** — same uncovered HARD-GATE violation as Round 1 (CAP-SEC-001, Critical); no exception records exist.

## 3. Critical rules (explicit re-verification — command step 9)

### CAP-SEC-001 — Model authorization explicitly for every exposed service — **FAIL** — Critical *(HARD-GATE)*
- **Authority:** SAP-REQ · **Runtime:** Both (Node.js applies) · **CAPire:** `guides/security/authorization` — **CURRENT**, L2 fetch 2026-08-12, statement re-confirmed verbatim ("By default, CDS services have no access control … without authorization modeling, authenticated users have access to all entities").
- **Detection re-run (not inferred):**
  - Enumerated `service` definitions in the current working tree of `srv/**/*.cds` → still two services: `loadbuilderV4` at `/AdminV4` (`srv/interaction_srv.cds` L4–L7) and `loadbuilder` at `/Admin` (L9–L172, ~78 exposed entities + 7 operations).
  - Searched the entire tree (`srv/`, `db/`, `app/`) for `@requires` / `@restrict` / `restrict:` — **zero matches** (repo-wide grep result: none). No `srv/access-control.cds` was created; no annotate file was added.
  - XSUAA `Admin`/`User` scopes and role templates in `xs-security.json` remain unchanged and remain **unreferenced by the model**.
  - Route configuration (`app/xs-app.json`) and MTA topology (`mta.yaml`) are unchanged.
- **Verdict:** FAIL. Detection procedure did not pass; nothing was inferred from file changes elsewhere.
- **Risk (current operational — LIVE):** unchanged from Round 1 — any authenticated user has full generic CRUD on all exposed entities (`Variants.Owner`, `Config.Destination`, all S/4 replica data, all custom operations).
- **Remediation status:** **Not applied** between rounds.

The two other applicable HARD rules in the M3 set were explicitly re-verified:
- **CAP-SEC-011** — full re-walk performed (§7) → PASS (with same caveat carried).
- **CAP-SEC-018** — NOT APPLICABLE (`mcp: false` in the profile — unchanged).

## 4. Defects — SOFT-gate rule violations (re-run detection)

Every SOFT FAIL was re-checked against its detection guidance on the current working tree. Consolidated root-finding blocks in §15; per-rule evidence below (per report template §3, per-rule verdicts retained).

### CAP-ARCH-003 *(supporting)* — Design services for single use cases — FAIL — High
- Detection re-run: `loadbuilder` still exposes ~78 of 88 domain entities across mixed concerns to all six UI apps; no service split occurred. **FAIL — unchanged from Round 1.**

### CAP-SRV-001 — Expose use-case projections, not persistence entities — FAIL — High
- Detection re-run: every service entity in `srv/interaction_srv.cds` is still a bare `as projection on custom.X` with zero `excluding`/column selection; `Variants.Owner` and `Config.Destination` still exposed. **FAIL — unchanged.**

### CAP-SRV-002 — Rely on generic providers — FAIL — High
- Detection re-run under the **updated Detection guidance (SCP-1)**: enumerated all `srv.on/before/after` sites and applied the new step 3 exemption.
  - Remote-backed entities with delegation handlers forwarding `req.query` → **57 sites** in `srv/interaction_srv.js` (e.g. L89, L173, L695, L732, L910, L967…) — **not flagged** per the documented integration pattern (`guides/services/consuming-services` verbatim: `this.on('READ', 'BusinessPartners', req => bupa.run(req.query))`). The exemption behaves exactly as calibration intended.
  - Local entities with query-discarding handlers still FAIL: `Variants` READ at L608–L618 reconstructs a `SELECT` and drops `req.query` semantics (`$filter`/`$top`/`$skip`/`$search` silently lost); `CriticalityConfig` READ at L540 reimplements sorting in memory though `$orderby` covers it. **FAIL — unchanged.**

### CAP-SRV-003 — Prefer declarative techniques — FAIL — Medium
- Detection re-run: still no `@readonly` anywhere; ownership filter still imperative in the same L620–L632 block; `managed` still not used for CreatedAt. **FAIL — unchanged.** Consolidated into Root 2 (ownership aspect) — see §15.

### CAP-SRV-005 — OData V4; V2 only as documented legacy — FAIL — High *(version-sensitive: L3 re-verified)*
- Detection re-run: `package.json` still lists **`@sap/cds-odata-v2-adapter-proxy` ^1.9.21** (deprecated predecessor); the currently documented package `@cap-js-community/odata-v2-adapter` (CAPire `guides/protocols/odata`, L3 fetch 2026-08-12) is not adopted; no ADR/comment justification was recorded anywhere. **FAIL — unchanged.**

### CAP-SRV-008 — Optimistic concurrency decision — FAIL — Medium *(CONCURRENT-EDIT)*
- Detection re-run under the calibrated profile: `concurrent_edit: true # source: repo-evidence` now unambiguously establishes applicability; the rule's detection step 3 states: *"Neither annotation nor a documented concurrency decision → FAIL per entity."* No `@odata.etag` anywhere; no LWW decision recorded. **FAIL.**
- **Round 1 verdict-shift note:** Round 1 marked SRV-008 NOT ASSESSABLE in §6 while §8 already read FAIL — an internal Round 1 report inconsistency the current re-run resolves. This is a Round 1 **report application** defect, not a rule/workflow defect (calibration-item type: report execution). See §19 calibration item C-4. No standard change proposed.

### CAP-SEC-010 — Instance-based authorization declaratively — FAIL — High *(INSTANCE-AUTH)*
- Detection re-run: Variants row-auth still in handlers only (L608–L632); **UPDATE path still entirely unguarded** (no UPDATE handler for Variants in the current file; `grep 'UPDATE'` shows only `RPItems.drafts`, `PurchaseReqnItemText`, `RP` — Variants absent); DELETE check still lets any user delete `IsGlobal` variants. No `@restrict` on Variants. **FAIL — unchanged.** Consolidated into Root 2.

### CAP-SEC-012 — Validate externally writable input declaratively — FAIL — High
- Detection re-run: still no `@assert.*` anywhere; `CriticalityConfig` UPDATE path still bypasses the range/sequence checks (only CREATE guarded, L553); `getBulkATP` / `getUomConversion` / `getMaterialAtpDetails` still take stringly-typed JSON parameters. **FAIL — unchanged.** Active-entity draft caveat statement re-confirmed verbatim from `guides/uis/fiori` (L2, 2026-08-12).

### CAP-SEC-013 *(supporting, newly in-scope this round per calibration SCP-3)* — Injection-safe queries — FAIL — Critical *(SUPPORTING at M3 — non-gating; owning milestone M4)*
- **Applicability trigger:** M3 SUPPORTING condition `CUSTOM-OPS ∧ (REMOTE ∨ MASHUP)` is satisfied by the profile (`custom_operations: true` ∧ `remote_services: true` ∧ `mashups: true`) — matrix footnote ³ fires.
- **Detection re-run:** `srv/interaction_srv.js` builds remote OData URLs by string-interpolating client-controlled `Material` and `Plant` values into query text at **three sites**: L3641 (getBulkATP direct path), L3674 (getBulkATP fallback), L3799 (getMaterialAtpDetails) — pattern verbatim: ``/CalculateAvailabilityTimeseries?Material='${Material}'&SupplyingPlant='${Plant}'&ATPCheckingRule='A'``. Per SEC-013 statement: *"User input MUST enter queries only as parameter values, never as query text or structure"* — the runtime the query text targets is a remote SAP OData service, not the local DB, but the rule's Node.js statement (`cds-ql`: "Never use string concatenation when constructing queries!" — L2 fetched 2026-08-12, verbatim) is unambiguous.
- **Non-gating at M3** (matrix §1.1 clarification) — reported and escalated in §5.1, but does not by itself gate this milestone; the HARD gate applies at CAP-SEC-013's PRIMARY milestone M4 and FINAL-GATE M6. **FAIL.**
- **Note:** this is the same evidence that appeared in Round 1's `§5 Cross-cutting security observation` — the calibration mapping change (SCP-3) now brings it in-scope as a proper rule finding with its own §8 row rather than only a §5 observation. R6 walkthrough → observed as intended.

### CAP-SEC-014 *(supporting)* — Request-flooding limits — FAIL — Medium
- Detection re-run under the **updated Detection guidance (SCP-2)**: Node.js `cds.odata.batch_limit` is not set in `package.json`/`.cdsrc.json` (verified in the current tree); no `$expand` guard in `before` handlers; no rate-limiting decision. Pagination cap `@cds.query.limit.max: 10` remains explicitly decided but anomalous (see §5 observation carried forward). **FAIL — unchanged**; the new Node batch-limit check fires as expected but does not change the verdict.

## 5. Recommendations & observations (non-defects)

### 5.1 Security concerns — cross-cutting security observations

- **The Round 1 cross-cutting security observation on the injection pattern is RESOLVED to a proper rule finding** in this round (§4 CAP-SEC-013). Owning rule + milestone unchanged (CAP-SEC-013 · M4 PRIMARY / M6 FINAL-GATE). Status: `carried forward → promoted to §4 finding` at Round 2 via calibration SCP-3 (matrix change). Immediate remediation recommendation restated: parameterize the remote query (use CAP's `cds.ql`-style parameter binding or a positive-list check on `Material`/`Plant`).
- No new cross-cutting observations discovered this round.

### 5.2–5.4 Other observations

- `@cds.query.limit.max: 10` on `loadbuilder` still anomalous — capped-generic-queries observation carried from Round 1 (A-3).
- `testPurchaseOrder()` still declared without a handler (dead API surface).
- `getBulkATP`/`getUomConversion` still stringly typed — noted, remediation is SEC-012's territory.
- Newly noted, no defect: an additional remote service `ZSB_MRPCONTROLLER_Private` was wired in `package.json` between rounds (destination `F4D_400_Addtional_auth`). It **broadens** the SEC-001 exposure surface and reinforces (does not weaken) the current gate FAIL.

## 6. NOT ASSESSABLE rules

### CAP-PERF-001 — Reliable pagination — NOT ASSESSABLE (unchanged from Round 1)
`workload.concurrent_paging: unknown` — owner input has not arrived between rounds. Rule stays NOT ASSESSABLE naming the flag (workflow behavior per WCP-3: never silently NOT APPLICABLE, never guessed). **Needed:** owner confirmation of the paging-under-concurrent-modification scenario.

## 7. Applicability decisions

CAP-SRV-009 (media ✗), CAP-SEC-018 (mcp ✗) — NOT APPLICABLE per profile (unchanged).

**RETROSPECTIVE SUPPORTING selection (per calibration WCP-1):** bounded to the M3 checklist's supporting list, per-row rationale recorded:

| Supporting row | Criterion (a) profile capability / (b) evidence exists / (c) related PRIMARY finding | Selected? |
|---|---|---|
| CAP-ARCH-003 | (b) 2 services with ~78 exposed entities present + (c) CAP-SRV-001 finding names it | **Yes** |
| CAP-CDS-007 | (b) `app/services.cds` + per-app annotation files present in `app/*/webapp/` | **Yes** |
| CAP-SEC-014 | (b) `@cds.query.limit.max: 10` present at `srv/interaction_srv.cds:9` (an explicit decision-artifact of this rule) | **Yes** |
| CAP-ERR-001 | (b) 43 CAP error-API sites in `srv/interaction_srv.js` | **Yes** |
| CAP-SEC-013 (new — conditional M3 supporting, matrix footnote ³) | (a) `custom_operations: true` ∧ `remote_services: true` — condition met | **Yes** |
| CAP-ERR-005 | — (matrix Round-1-calibration change ⁴ removed this appearance) | **Not in list** |

Round 1 → Round 2 SUPPORTING delta: **ERR-005 out, SEC-013 in** — driven by the matrix mapping change (SCP-3 + ERR-005 calibration), NOT by evidence changes in the project.

## 8. Full per-rule results

| Rule | Class | Gate | Verdict | Δ vs R1 | Evidence pointer |
|---|---|---|---|---|---|
| CAP-SRV-001 | PRIMARY | SOFT | **FAIL** | same | `srv/interaction_srv.cds` — zero exclusions across ~78 projections |
| CAP-SRV-002 | PRIMARY | SOFT | **FAIL** | same (57 delegation sites correctly exempted under SCP-1) | `srv/interaction_srv.js` L608 (Variants — query discarded), L540 (CriticalityConfig) |
| CAP-SRV-003 | PRIMARY | SOFT | **FAIL** | same | no `@readonly` model-wide; ownership handled imperatively |
| CAP-SRV-004 | PRIMARY | SOFT | PASS | same | actions/functions modeled correctly (obs: `testPurchaseOrder` still unimplemented — §5) |
| CAP-SRV-005 | PRIMARY | SOFT | **FAIL** | same | deprecated adapter still in `package.json`; no justification |
| CAP-SRV-006 | PRIMARY | SOFT | PASS | same | protocol exposure decisions still visible (explicit `@path`, proxy explicitly wired) |
| CAP-SRV-007 | PRIMARY | SOFT | PASS | same | `@odata.draft.enabled` on 6 entities; no hand-built draft mechanisms |
| CAP-SRV-008 | PRIMARY | SOFT | **FAIL** | **NA → FAIL** (Round 1 §6/§8 inconsistency resolved via strict detection re-run; see C-4) | `concurrent_edit: true` decided; no `@odata.etag`, no LWW decision |
| CAP-SRV-009 | PRIMARY | SOFT | NOT APPLICABLE | same | `media: false` |
| CAP-SEC-001 | PRIMARY | **HARD** | **FAIL** | same | §3 — repo-wide grep confirms zero `@requires`/`@restrict` |
| CAP-SEC-010 | PRIMARY | SOFT | **FAIL** | same | Variants row-auth in handlers; UPDATE unguarded |
| CAP-SEC-011 | PRIMARY | **HARD** | PASS | same (re-walked — §7) | no stricter-restricted targets exist |
| CAP-SEC-012 | PRIMARY | SOFT | **FAIL** | same | no `@assert.*`; UPDATE bypasses CriticalityConfig checks |
| CAP-SEC-018 | PRIMARY | **HARD** | NOT APPLICABLE | same | `mcp: false` |
| CAP-PERF-001 | PRIMARY | SOFT | NOT ASSESSABLE | same | §6 — `concurrent_paging: unknown` |
| CAP-ARCH-003 | SUPPORTING | SOFT | **FAIL** | same | §4 |
| CAP-CDS-007 | SUPPORTING | SOFT | PASS | same | UI annotations factored in `app/services.cds` |
| CAP-SEC-013 | SUPPORTING (new via SCP-3) | HARD at M4/M6; **non-gating at M3** | **FAIL** | **new appearance** — same evidence as R1's cross-cutting observation | L3641, L3674, L3799 — string-interpolated remote query text |
| CAP-SEC-014 | SUPPORTING | SOFT | **FAIL** | same (detection expanded per SCP-2; verdict unchanged) | no Node `batch_limit`, no expand guard, no rate-limiting decision |
| CAP-ERR-001 | SUPPORTING | SOFT | PASS | same | 43 CAP error-API sites; the sole raw `throw` (L4959) is an internal config failure |
| CAP-ERR-005 | — (removed from M3 supporting per calibration ⁴) | — | — | — | Out of M3 scope in Round 2 (M4 primary/UI-conditional remains) |

## 9. Exceptions honored

None on file — the project still has no exception records (`docs/capm/exceptions/` still absent; no `docs/` directory). No ORG-PENDING rules in the evaluated set.

## 10. Standards & CAPire evidence verification

**Re-verified this round (not silently reused from Round 1).** All fetches performed 2026-08-12.

| Source (cap.cloud.sap/docs/…) | Rules | Level | Status |
|---|---|---|---|
| guides/security/authorization | SEC-001, SEC-010, SEC-011 | **L2** | **CURRENT** — "no access control by default"; Node target-only evaluation; `@restrict where $user` all re-confirmed |
| guides/services/constraints | SEC-012, SRV-003 | **L2** | **CURRENT** — "@mandatory checked", "enforced automatically … request is ultimately rejected and the transaction rolled back" re-confirmed |
| guides/uis/fiori | SRV-007, SEC-012 | **L2** | **CURRENT** — draft-bypass caveat re-confirmed verbatim |
| guides/security/data-protection | SEC-014, SEC-013 | **L2** | **CURRENT** — Node `cds.odata.batch_limit`, Node `$expand` custom handler, rate limiting, prepared statements + positive-list validation all re-confirmed |
| node.js/cds-ql | SEC-013 | **L2** | **CURRENT** — "Never use string concatenation when constructing queries!" and "Never surround tagged template strings with parentheses!" re-confirmed verbatim |
| guides/protocols/odata | SRV-005 | **L3** | **CURRENT** — "OData V2 is deprecated"; `@cap-js-community/odata-v2-adapter` package name re-confirmed |
| guides/services/served-ootb | SRV-002, SRV-008, PERF-001 | **L3** | **CURRENT** — generic CRUD, reliable-paging keys, "only for OData V4", `@odata.etag`/412 re-confirmed |
| guides/services/providing-services | SRV-001, ARCH-003 | L2 | **CURRENT** — 1:1 warning + "single use cases" re-confirmed |
| guides/services/consuming-services | SRV-002 SCP-1 clarification | **L2** | **CURRENT** — `req => bupa.run(req.query)` delegation pattern re-confirmed verbatim |
| node.js/events | ERR-001 | L1 | CURRENT |
| cds/aspects + guides/domain/ | CDS-007 | L1 | CURRENT |

Version register (`docs/version-management.md`) consulted 2026-08-12 (still within cadence) for SRV-005/PERF-001. **Governance flags: none this round** — the SCP-2 mechanism-list evolution that Round 1 flagged is now reflected in the rule text (calibration `4fbaac7`), so it is no longer a live governance note.

## 11. Remediation plan

**No remediation was applied between Round 1 and Round 2**, so all Round 1 remediation items carry forward unchanged. The ordering and content are identical to [Round 1 §11](../pilot-1-scs-m3/review-report.md#11-remediation-plan) (items 1–11). Not repeated here; the pointer is the plan.

## 12. Outstanding risks & next-milestone readiness

Risks unchanged from Round 1 (all Round 1 findings persist as current operational risk on this LIVE application). **M4 entry criteria NOT met** (M3 gate FAIL). Next steps in `/capm-review-milestone re-review M3` order once real remediation lands: SEC-001 → SEC-010/SRV-003 (Root 2) → SEC-011 mandatory re-walk after SEC-001 (introduces restriction differentials) → the SOFTs → SEC-013 (which will also lift the M3 supporting finding).

---

# Round 1 ↔ Round 2 Analysis (Step 18)

| Dimension | Round 1 | Round 2 | Change |
|---|---:|---:|---|
| Applicable rules (selected M3 set) | 20 | 20 | 0 |
| PRIMARY selected | 15 | 15 | 0 |
| SUPPORTING selected | 5 | 5 | ERR-005 out, SEC-013 in |
| PASS | 6 | 6 | 0 |
| FAIL | 11 | 11 | 0 |
| NOT APPLICABLE | 2 | 2 | 0 |
| NOT ASSESSABLE | 1 | 1 | 0 |
| HARD failures | 1 (SEC-001) | 1 (SEC-001) | 0 |
| Critical failures | 1 (SEC-001) | 1 (SEC-001) | 0 |
| Root findings (executive summary count) | not consolidated | 7 | new — consolidation convention now applied |
| Cross-cutting security observations | 1 new (injection) | 0 new (Round 1's promoted to a §4 rule finding via SCP-3) | mechanism worked as designed |
| CAPire sources re-verified | 12 | 11 | (Round 1 also verified `guides/services/custom-actions` and `custom-code`; Round 2 re-verified in the shared same-day pool) |

## Interpretation

The verdict counts are **identical** between rounds because **no remediation happened**: the last commit on the project's default branch is still `c1f51ac` (2026-07-06), and the working-tree changes since Round 1 are UI code, a new remote-service wiring (adds surface, doesn't remove it), and cosmetic `db/interactions.cds` whitespace. Not a fix in the codebase. The re-review branch of the workflow behaved exactly as calibration required: it **re-ran detection for every previously-FAILED and NOT-ASSESSABLE rule + touched files + the mandatory SEC-011 re-walk**, and did **not** infer remediation success from any change. On the standard-side, the calibration changes took effect visibly:

- **SCP-1 (SRV-002)** worked: 57 legitimate remote delegations were exempted; the two genuine local violations still fire. No under- or over-fire.
- **SCP-2 (SEC-014)** worked: the Node batch-limit check now runs as part of detection; verdict unchanged because the value is still absent — confirming the mechanism update was truly explanatory, not a security-intent change.
- **SCP-3 (SEC-013 conditional M3 supporting)** worked: Round 1's cross-cutting observation was promoted to a proper §4 rule finding with its own §8 row and CAPire verification; non-gating at M3 as designed; HARD gate intact at M4/M6.
- **WCP-1 (retrospective SUPPORTING selection)** worked: 5 selected, bounded to the checklist list, per-row (a)/(b)/(c) rationale recorded — ERR-005 correctly not selected.
- **WCP-3 (workload `unknown`)** worked: PERF-001 stayed NOT ASSESSABLE naming `concurrent_paging`, not silently NA; profile validation confirmed the `concurrent_edit: true # source: repo-evidence` note without contradiction.
- **Finding-consolidation convention** worked: 7 root findings in §15, per-rule verdicts retained in §8.
- **Review-mode + LIVE wording** worked: header shows RETROSPECTIVE + "findings represent current operational risk"; report reads unambiguously.

The one **verdict-level change** — CAP-SRV-008 NOT ASSESSABLE → FAIL — is a Round 1 report application defect, not a rule/framework change (see C-4).

---

## 15. Root-finding consolidation (per report-template §3 convention)

The Round 2 executive summary counts the seven roots below (§2's per-rule counts stay per-rule):

- **Root A — No service-level authorization model** — FAIL — Critical. Affected rules: CAP-SEC-001 (Critical/HARD). Evidence per §3. Root cause: authorization not modeled anywhere in `srv/*.cds` despite XSUAA scopes existing.
- **Root B — Variants ownership check bypassable on write paths** — FAIL — High. Affected rules: CAP-SEC-010 (High/SOFT), CAP-SRV-003 (Medium/SOFT — declarative alternative ignored). Root cause: row-level access implemented in handlers with UPDATE entirely unguarded and global-variant DELETE unconstrained; the declarative `@restrict` form was never used.
- **Root C — Whole-model 1:1 API exposure** — FAIL — High. Affected rules: CAP-SRV-001 (High/SOFT); reinforced by CAP-ARCH-003 (High/SOFT — service breadth) and CAP-SRV-003 (Medium/SOFT — missing `@readonly`). Root cause: bare projections across ~78 entities.
- **Root D — Deprecated protocol + deprecated adapter package, undocumented** — FAIL — High. Affected rules: CAP-SRV-005 (High/SOFT). Stand-alone (SRV-006 passes: protocol exposure is deliberate).
- **Root E — Input validation not declarative; UPDATE path unvalidated** — FAIL — High. Affected rules: CAP-SEC-012 (High/SOFT). Stand-alone.
- **Root F — Query-degrading handlers on local entities** — FAIL — High. Affected rules: CAP-SRV-002 (High/SOFT). Stand-alone.
- **Root G — Undecided limits + injection-pattern in remote-query construction** — FAIL — Medium *(and Critical for the injection at its owning milestone)*. Affected rules: CAP-SEC-014 (Medium/SOFT — limits undecided), CAP-SEC-013 (Critical, non-gating at M3, HARD at M4). Root cause: the request-amplification/DoS surface and the query-construction hygiene are undecided/uncoded — same architectural gap.
- Independently: CAP-SRV-008 (concurrency decision missing — Medium/SOFT) — a stand-alone defect, not consolidated with the others.

Per-rule verdict counts remain 6/11/2/1 (§2). Executive defect count = **7 roots + SRV-008 = 8 distinct defects**.

---

# Phase 4 Round 2 Calibration (Steps 19–20)

## Calibration pass

| # | Class | Rule | Explanation |
|---|---|---|---|
| C-4 | **Round 1 report execution defect** (not rule, not workflow) | CAP-SRV-008 | Round 1 §6 marked NOT ASSESSABLE while §8 already read FAIL — a self-inconsistency in the Round 1 report. Round 2 re-runs detection under a profile whose `concurrent_edit: true` source is now explicit and reaches FAIL cleanly. Same failure-mode class as Phase 3's SRV-001 false negative in the Node fixture report: the traceability chain finds report-application defects when detection is actually re-run. **No standard/workflow change proposed.** |
| — | **False positives** | — | None. The SEC-013 finding was already validated as real in Round 1 (only its scope was extended). SRV-002's 57 delegation sites were correctly exempted by SCP-1. |
| — | **False negatives** | — | None new. Round 1's identified negative (injection at M3) is now caught by the calibration mapping (SCP-3) — the mapping change effectively converts the FN into a proper finding. |
| — | **Noise** | — | ERR-005 no longer appears at M3 — the noise item Round 1 identified is resolved. No new noise. |
| — | **Ambiguities** | — | A-1 (workload flags need owner input) still open but now correctly parked at NOT ASSESSABLE rather than mis-filtered. A-2 (SRV-001 detection vs replica architectures) still open — one data point; deferred to a future pilot. A-3 (`@cds.query.limit.max: 10` anomaly) still an observation. **No new ambiguities.** |
| — | **Missed evidence** | — | None discovered. |
| — | **Unexpected behavior** | — | None. Every calibration change produced the effect it was designed to produce (see interpretation above). |

## Round 2 pilot classification: **AMBER**

**Justification.** On workflow behavior alone this round is unambiguously GREEN: every Round 1 calibration item behaves exactly as designed (SCP-1/2/3, WCP-1/3, consolidation convention, review-mode header, `unknown` handling); the mandatory SEC-011 re-walk was performed; detection was re-run on every previously-failed rule + touched files; no rule text needed to change; no false positives; no new false negatives; the one verdict-level flip (SRV-008) is a legacy report execution defect, not a framework defect. **AMBER rather than GREEN because the pilot's stated *closed-loop* objective — REVIEW → FINDINGS → ACTUAL REMEDIATION → RE-REVIEW → GATE — is only half-exercised.** No remediation happened between rounds on the real project; the re-review branch of the loop was validated (correctly reports "still FAIL", correctly refuses to infer fixed-ness from unrelated file changes), but the *remediation-then-detection-passes* branch remains untested on a real codebase (Phase 3's fixtures did test it, on constructed fixtures). Closing that half of the loop needs one of: (a) a subset of Root A/B/C to actually land in scs and be re-reviewed; (b) a second real project pilot where remediation is genuinely done. This is an observation about pilot inputs, **not** a workflow defect.

---

# Governance notes

## Developer participation (Step 15)

**Developer participation not available.** No project developer was involved in Round 2. No feedback fabricated. Round 2 continues to satisfy Steps 12–14 of the pilot brief on the reviewer-side alone; developer experience remains "Not assessed" as it was in Round 1.

## Production-proven claim (Step 21)

**Not claimed.** In the language of Step 21: *"The CAPM Development Standards workflow has completed a controlled Round 2 M3 re-review on this application; the closed loop is partially validated (re-review branch confirmed; remediation branch pending real-project remediation)."* The standard still requires organizational adoption, remediation-branch validation on a real project, and broader project experience before any "production-proven" language.

## Not modified (Step 22)

The review was read-only. The only file written to the project is `capm-profile.yaml` (its Round 1 form updated during the Round 1 calibration to reflect the new workload semantics — that update landed before Round 2 started, and Round 2 wrote nothing to the project).

---

*Scope statement: this Round 2 report evaluates only the applicable M3 standards listed in §8 at standard version `4fbaac7`. It makes no claim about overall CAP best-practice compliance, production readiness, or any other milestone.*
