# Phase 4 — Round 1 Calibration Record (2026-08-12)

Controlled resolution of [Pilot 1](pilot-1-scs-m3/review-report.md)'s calibration findings, executed **before** Round 2. Every change below was CAPire-re-verified where it touches SAP-sourced content; workflow changes are organizational/process policy and are never presented as SAP authority.

## Finding classification (task objective)

| Pilot finding | Classification | Disposition |
|---|---|---|
| SCP-1 SRV-002 federation over-fire risk | Genuine standard defect (detection guidance gap) | **ACCEPT** |
| SCP-2 SEC-014 stale Node `$batch` mechanism | Genuine standard defect (explanatory/mechanism note only — statement intent unaffected) | **ACCEPT** |
| SCP-3 SEC-013 invisible at M3 | Workflow defect (milestone mapping gap) | **ACCEPT WITH MODIFICATION** (Option B) |
| WCP-1 retrospective SUPPORTING selection | Workflow defect (undefined procedure branch) | **ACCEPT** |
| WCP-3 workload flags not establishable | Workflow defect (profile evidence model gap) | **ACCEPT** |
| SRV-003 duplicating SEC-010 | Report presentation issue | **ACCEPT** (reporting convention; rules unchanged) |
| ERR-005 noise at M3 | Workflow defect (milestone fit of a supporting appearance) | **ACCEPT WITH MODIFICATION** (mapping change only) |
| Retrospective vs LIVE wording | Report presentation issue | **ACCEPT** (terminology, header field) |
| SRV-005/006 distinctness | Item that should remain unchanged | **NO CHANGE** — conclusion recorded below |
| A-2 SRV-001 detection vs replica architectures | Project-specific observation → possible detection note | **DEFER** to Round 2 (single data point) |
| A-3 `@cds.query.limit.max: 10` anomaly | Project-specific observation | **NO CHANGE** (rule stays decision-focused; report observation covered it) |

## Decisions in detail

### SCP-1 — CAP-SRV-002 federation exemption — ACCEPT (location: **B. Detection guidance**)
The rule *statement* was never wrong (delegation handlers implement a genuine domain delta: remote federation); the *detection* would over-fire on a naive step-2 scan. Correction placed in Detection guidance (new step 3) + one Evidence-expected sentence + a SAP-reference addition — **exactly as far as CAPire supports and no further**: the exemption covers only handlers delegating to remote services, and they must still forward/deliberately map `req.query` (a delegation handler discarding the client query remains a FAIL — verified against Pilot 1: the `Variants` handler still fails, all 57 `run(req.query)` delegation sites pass). Authority (SAP-REC), severity (High), statement, and intent unchanged.
**CAPire:** `guides/services/consuming-services` — CURRENT, verified 2026-08-12 — "Write a handler function to delegate a query to the remote service and run the incoming query on the external service"; `this.on('READ', 'BusinessPartners', req => bupa.run(req.query))`. (The pre-restructure `guides/using-services` URL is 404 — the source map already carries the current URL.)

### SCP-2 — CAP-SEC-014 mechanism update — ACCEPT (smallest correction)
Re-verified `guides/security/data-protection` (CURRENT, 2026-08-12), verbatim: Node.js `$batch` **must** be configured with `cds.odata.batch_limit = <max_requests>`; Java `cds.odataV4.batch.maxRequests` / `cds.odataV2.batch.maxRequests`; Node.js `$expand` unchanged ("CAP applications have to limit the amount of `$expands` per request in a custom handler"); rate limiting unchanged ("Applications need to establish an adequate rate limiting strategy"). **Only the mechanism parenthetical was stale**, not the rule. Changed: statement parenthetical, CAP-version note, detection step 2, one Good-example block. Authority stays SAP-REQ, severity stays Medium, security intent untouched. Pilot 1's SEC-014 verdict is unaffected (decisions were absent under either mechanism list).

### SCP-3 — CAP-SEC-013 at M3 — ACCEPT WITH MODIFICATION: **Option B**
- **Option A (M4 only):** rejected — Pilot 1 produced a real injection pattern at M3 that only survived via an ad-hoc observation; "wait for M4" loses early, cheap detection on exactly the projects with the largest surface.
- **Option C (move PRIMARY earlier):** rejected — M3 is the service-contract milestone; handlers (SEC-013's main evidence) are M4's subject. Moving primary ownership would demand evidence that mostly doesn't exist yet in greenfield flow and would break the milestone intent + M6 FINAL-GATE structure.
- **Option B (M4 PRIMARY + conditional M3 SUPPORTING):** accepted. Condition — observable profile characteristics, NOT "all M3 projects": **`CUSTOM-OPS ∧ (REMOTE ∨ MASHUP)`** (custom operations exist and remote services/mashups are consumed — the combination where handler-built queries/URLs appear before M4, which is precisely Pilot 1's shape). Non-gating at M3 (matrix §1.1 clarification); violations escalate via the cross-cutting security observation mechanism; HARD gate unchanged at M4 PRIMARY and M6 FINAL-GATE. No duplication with SEC-001 (authorization modeling) or SEC-010 (instance restrictions) — SEC-013 owns query/URL construction hygiene.
**CAPire:** no rule-text change → rule sources untouched; mapping is workflow policy (no SAP authority claimed).

### WCP-1 — Retrospective SUPPORTING selection — ACCEPT
Two review modes defined (profile spec + command step 3/5 + review model): **DEVELOPMENT** keeps the existing changes-touch-subject criterion (fixture behavior unchanged — F6 re-run PASS); **RETROSPECTIVE** uses evidence-driven selection **bounded to the milestone checklist's supporting list**: select a row iff (a) the profile declares the driving capability, or (b) the rule's *Evidence expected in code* artifacts exist in the repository, or (c) a PRIMARY rule of this review produced a finding listing it under *Related rules*. Per-row selected/not-selected rationale with the criterion is recorded in report §7. Never all-by-default, never outside the list. Organizational policy — no SAP authority.

### WCP-3 — Workload flag evidence states — ACCEPT
`workload` flags now take `true | false | unknown` with an optional `# source:` note (`owner-declared (<who>, <date>)` / `repo-evidence (<pointer>)`). Semantics: **declaration = hypothesis** (reviewers cross-check; contradiction → CONTRADICTORY profile finding, review stops); repository evidence can confirm/establish; `unknown` → dependent rules NOT ASSESSABLE naming the flag; Claude never sets a workload value by inference (propose-and-accept only). Deliberately minimal: one extra allowed value + a comment convention — no new blocks, no per-flag sub-objects. The scs project profile was migrated (4 flags → `unknown`); the pilot artifact copy stays as the historical record. Organizational policy — no SAP authority.

### Finding consolidation — ACCEPT (reporting convention only; rules NOT merged)
Defined in the report template §3 + command step 13: same underlying defect found by multiple rules → ONE `Root finding:` block naming the root cause, listing all affected rule IDs with each rule's own evidence; per-rule verdicts stay in §8; the defect counts once in §2's executive summary. Pilot example re-rendered (R5): *"ownership check bypassable" — affected: CAP-SEC-010, CAP-SRV-003*.

### ERR-005 at M3 — ACCEPT WITH MODIFICATION (mapping change, not severity)
Pilot evidence: the finding was valid (43 hardcoded English error sites) but not actionable at M3 — remediation is M4 handler work, and the M3 "error contract sketch" is ERR-001's subject (which stays M3 SUPPORTING). Resolution: **remove ERR-005's M3 SUPPORTING appearance** (matrix footnote ⁴); it remains M4 PRIMARY, UI-conditional, Medium, SOFT — severity was NOT changed to shorten reports; the milestone fit was corrected. Options "narrower detection" and "present as advisory" rejected: detection is accurate, and ADVISORY would understate a real Fiori-UX/localization defect at its owning milestone.

### Retrospective vs LIVE terminology — ACCEPT
Review-mode model (DEVELOPMENT / RETROSPECTIVE) + deployment overlay: `project.milestone: LIVE` ⇒ report states findings are **current operational risk**. One header field + one command-step determination — no new process. The four report questions (pre-completion? retrospective? deployed? historical-vs-current risk?) are all answerable from the header.

### Cross-cutting security observations — ACCEPT
Mechanism (command step 9a + template §5.1): strong evidence of a security issue owned by a later milestone's rule → recorded observation (owning rule ID, owning milestone, evidence, risk, immediate-remediation recommendation, status new/carried-forward); does NOT fail the current milestone unless the matrix maps the rule there; **carried into every subsequent review until the owning milestone evaluates it** (command step 3 loads prior reports' open observations). Decision on Critical-rule blanket visibility: **rejected** — all 11 Critical rules are already HARD + M9-visible (matrix §3), and this observation mechanism plus SEC-013's targeted conditional mapping cover the demonstrated gap; altering all Critical rules would be change without evidence.

### SRV-005 / SRV-006 — NO CHANGE (pilot conclusion recorded)
Pilot 1 settled the fixture-era question: the rules diverged on the same project (005 FAIL — deprecated protocol version, no recorded justification, deprecated adapter package; 006 PASS — exposure itself deliberate and visible). **Distinct failure modes; both remain PRIMARY at M3.** When both fail on one service for the same root cause, the finding-consolidation convention (above) explains their joint appearance — no further reporting rule needed.

## CAPire verification of this calibration (Part 12)

| Change | CAPire source | Authority | Runtime | Status | Verified |
|---|---|---|---|---|---|
| SCP-1 (SRV-002 detection) | cap.cloud.sap/docs/guides/services/consuming-services | SAP-REC (unchanged) | Both | **CURRENT** | 2026-08-12 |
| SCP-2 (SEC-014 mechanisms) | cap.cloud.sap/docs/guides/security/data-protection | SAP-REQ (unchanged) | Both (mechanisms differ) | **CURRENT** | 2026-08-12 |
| SCP-3, WCP-1, WCP-3, consolidation, ERR-005 mapping, review modes, observations | — (milestone mapping / process policy) | ORG process policy — **no SAP authority claimed or implied** | — | n/a | 2026-08-12 |

## Validation impact (Part 14)

Scenario catalog 36 → **44** (P9–P10, F7–F8, E6, R5–R7). Executed 2026-08-12: **8/8 new PASS; 4 affected re-runs PASS (P1, P2, F6, G3); 32 unaffected by inspection** — full record in the [validation results addendum](../docs/validation-results-2026-08.md#addendum--round-1-calibration-execution-2026-08-12-post-pilot-1).

## Deferred / open items for Round 2

1. **A-2** — SRV-001 detection note for replica/cache architectures (defer: one data point; collect Round 2 evidence).
2. **O-1 + `unknown` workload flags** — owner input for scs (`concurrent_paging`, `mass_data`, `hana_large_volume`, `major_upgrade_in_scope`) and `deployment.multi_landscape` confirmation.
3. **Developer experience** — include a developer in Round 2 (Pilot 1: "Not assessed").
4. **ORG-PENDING ratification** — ARCH-007, CICD-003, OPS-003 (unchanged, pre-existing).
5. **Noise watch** — CAP-CDS-001 on legacy-shaped models (pre-existing Phase 4 observation).
