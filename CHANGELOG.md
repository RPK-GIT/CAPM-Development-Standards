# Changelog

Human-readable summary of substantive changes to CAPM Development Standards. Full history in git; this document highlights the major decisions and calibration improvements, not every commit.

Rule IDs are stable — never reused. Amendments always preserve history (see [rule-governance.md](docs/rule-governance.md)).

## v1.0.0 — 2026-08-12

**First finalized release.** Real-project closed-loop validation demonstrated.

### Framework — carried forward from Phases 1–4

- **134 formal Layer 2 rules** across 20 categories (ARCH, CDS, CICD, DB, DEP, ERR, EVT, EXT, INT, LOG, LOGIC, MT, OPS, PERF, PRIV, SEC, SRV, TEST, TXN, VER). Distribution: 40 SAP-REQ · 89 SAP-REC · 2 GEN · 3 ORG-PENDING · 11 Critical (all HARD).
- **Three-layer architecture** — Principles / Rules / AI rules — with an explicit [authority taxonomy](docs/authority-levels.md).
- **M0–M9 lifecycle** with per-milestone checklists ([milestones/](development/milestones/m0-requirements.md)) and a canonical [rule-milestone matrix](development/rule-milestone-matrix.md) — 34 HARD-gate · 94 SOFT-gate · 6 ADVISORY.
- **Two Claude Code commands** — [`/capm-develop`](.claude/commands/capm-develop.md) (implement against the standard) and [`/capm-review-milestone Mx`](.claude/commands/capm-review-milestone.md) (independent read-only review).
- **CAPire evidence protocol** ([reviews/capire-verification.md](reviews/capire-verification.md)) — three verification levels, six source statuses, mandatory traceability chain.
- **Machine-readable project profile** ([development/project-profile.md](development/project-profile.md)) — the single input that makes rule filtering decidable without prose parsing.
- **48 operational validation scenarios** ([docs/operational-validation.md](docs/operational-validation.md); execution record: [validation-results-2026-08.md](docs/validation-results-2026-08.md)) — 40 pre-calibration base + 8 added at Round 1 calibration; all executed under Node.js + Java fixtures + real-project pilot inputs. *(The historical execution record used a 36 → 44 accounting; the actual counts are 40 → 48. Ratio and coverage unchanged; see [operational-validation.md](docs/operational-validation.md) footer.)*

### v1.0.0 finalization

- **Governance documentation** — [rule-governance.md](docs/rule-governance.md), [exception-governance.md](docs/exception-governance.md), [org-ratification.md](docs/org-ratification.md), [re-verification-cadence.md](docs/re-verification-cadence.md).
- **Adoption package** — [adoption-guide.md](docs/adoption-guide.md), [developer-guide.md](docs/developer-guide.md), [reviewer-guide.md](docs/reviewer-guide.md), [executive-overview.md](docs/executive-overview.md), [adoption-boundaries.md](docs/adoption-boundaries.md).
- Forward-only anonymization of one destination name in [Pilot 1 Round 2 report](pilots/pilot-1-scs-m3-round-2/review-report.md) — no history rewrite; the finding itself is unchanged.

### Validated against

- **Node.js and Java fixtures** — [worked-example-m3/](examples/worked-example-m3/README.md), [worked-example-m3-java/](examples/worked-example-m3-java/README.md).
- **Pilot 1 (`scs` — real, deployed BTP application) — Rounds 1, 2, 3** — see the *Phase 4* section below and the pilot artifacts in [pilots/](pilots/pilot-1-scs-m3/README.md).

### Known limitations at v1.0

- Pilot breadth is narrow: one live Node.js project, one milestone deeply reviewed. Java-runtime coverage in live projects and multitenant SaaS scenarios are under-piloted.
- Three ORG-PENDING rules (CAP-ARCH-007, CAP-CICD-003, CAP-OPS-003) require formal organizational ratification — see [org-ratification.md](docs/org-ratification.md).
- Full boundaries: [adoption-boundaries.md](docs/adoption-boundaries.md).

---

## Phase 4 — Real-Project Pilots (July–August 2026)

### Round 3 — Controlled remediation experiment · commit `ca25030`

- **Result: GREEN.** Two genuine `FAIL → PASS` transitions on real code:
  - `CAP-SEC-001` (Critical HARD) — service-level authorization added via new `srv/access-control.cds` aspect file in `scs`.
  - `CAP-SEC-010` (High SOFT) — instance-based `@restrict` on `Variants` covering the horizontal-privilege gap on the UPDATE path.
- 9 unremediated FAILs correctly preserved (no false PASS).
- Mandatory `CAP-SEC-011` re-walk executed after SEC-001 remediation (restriction differentials).
- Fresh CAPire re-verification on 2026-08-12 (11 sources CURRENT; zero governance flags).
- Full report: [pilots/pilot-1-scs-m3-round-3/review-report.md](pilots/pilot-1-scs-m3-round-3/review-report.md).

### Round 2 — Re-review · commit `99332a6`

- **Result: AMBER** (closed loop half-validated — no remediation between Rounds 1 and 2).
- Every Round-1 calibration change verified working under a real re-review:
  - SCP-1 (SRV-002 delegation-handler exemption) — 57 legitimate remote delegations correctly exempted; 2 genuine local violations still fired.
  - SCP-2 (SEC-014 Node `$batch` mechanism update) — new detection step ran; verdict unchanged (value still absent), confirming the change was explanatory.
  - SCP-3 (SEC-013 conditional M3 SUPPORTING) — Round 1's cross-cutting observation promoted to a proper §4 rule finding.
  - WCP-1 (retrospective SUPPORTING selection) — 5 selected, bounded, per-row rationale recorded.
  - WCP-3 (workload `unknown`) — PERF-001 stayed NOT ASSESSABLE naming the flag.
- Resolved Round 1 report inconsistency (SRV-008 NA/FAIL mismatch across sections) via strict detection re-run.
- Full report: [pilots/pilot-1-scs-m3-round-2/review-report.md](pilots/pilot-1-scs-m3-round-2/review-report.md).

### Round 1 calibration · commits `04e266b`, `4fbaac7`

Controlled resolution of Pilot 1's calibration findings:

- **Rule detection amendments** (2 rules) — both with fresh CAPire evidence, statements and intent unchanged:
  - CAP-SRV-002 — delegation-handler exemption added to Detection guidance (SCP-1). Evidence: [guides/services/consuming-services](https://cap.cloud.sap/docs/guides/services/consuming-services).
  - CAP-SEC-014 — Node.js `cds.odata.batch_limit` mechanism added to statement + detection (SCP-2). Evidence: [guides/security/data-protection](https://cap.cloud.sap/docs/guides/security/data-protection).
- **Matrix mapping changes** (2 rows):
  - CAP-SEC-013 gained conditional M3 SUPPORTING (footnote ³) with `CUSTOM-OPS ∧ (REMOTE ∨ MASHUP)` trigger — closes a milestone-scoping false negative surfaced by Pilot 1.
  - CAP-ERR-005 M3 SUPPORTING appearance removed (footnote ⁴) — milestone-fit calibration, not severity change.
- **Workflow additions:**
  - Review modes DEVELOPMENT / RETROSPECTIVE / LIVE + report header field.
  - Retrospective SUPPORTING selection algorithm (bounded to the milestone checklist, criterion-based).
  - Workload evidence states — flags now take `true | false | unknown`; declaration vs repository-evidence distinction.
  - Finding-consolidation convention (`Root finding:` blocks; per-rule verdicts retained).
  - Cross-cutting security observations with carry-forward duty.
- **Validation suite:** grew 40 → 48 scenarios (historical record labels this 36 → 44; the actual counts are 40 → 48). Executed 8/8 new + 4/4 affected re-runs = 12/12 PASS; 32 unaffected by inspection.
- Full record: [pilots/round-1-calibration.md](pilots/round-1-calibration.md); execution addendum in [validation-results-2026-08.md](docs/validation-results-2026-08.md).

### Round 1 · commit `bd0e637`

- **Result: AMBER.** First real-project M3 review against `scs` — a live, deployed BTP application (Node.js, `@sap/cds` 7.9.5, HANA Cloud, XSUAA, CF/MTA, 6 Fiori apps, 20+ S/4 remote services).
- 20 rules selected (15 primary + 5 supporting); 1 Critical HARD FAIL (SEC-001), 10 SOFT FAILs, 1 NOT ASSESSABLE (PERF-001 — workload flag unknown).
- Zero false positives; one false negative caught by cross-cutting observation (SEC-013 at M3, M4-owned).
- Framework performed reliably; five calibration items produced the Round 1 calibration above.
- Full report: [pilots/pilot-1-scs-m3/review-report.md](pilots/pilot-1-scs-m3/review-report.md).

---

## Phase 3 — Lifecycle Integration & Operational Validation (August 2026)

- **Prompt 1** — 134 rules mapped to M0–M9 via [rule-milestone-matrix.md](development/rule-milestone-matrix.md): PRIMARY / SUPPORTING / FINAL-GATE / CONDITIONAL applicability × HARD / SOFT / ADVISORY gate class. Five-state milestone results (`PASS` / `PASS WITH EXCEPTIONS` / `FAIL` / `NOT READY` / `NOT APPLICABLE`). Full Critical-coverage report. ORG rules explicitly `ORG-PENDING`.
- **Prompt 2** — operationalization: [`/capm-develop`](.claude/commands/capm-develop.md), [`/capm-review-milestone`](.claude/commands/capm-review-milestone.md), [machine-readable profile](development/project-profile.md), [CAPire verification protocol](reviews/capire-verification.md), remediation and re-review workflows, [Node.js worked example](examples/worked-example-m3/README.md). Templates: [review report](templates/review-report-template.md), [completion report](templates/completion-report-template.md), [remediation plan](templates/remediation-plan-template.md).
- **Prompt 3** — validation: 36 scenarios executed against Node.js + Java fixtures ([Java example](examples/worked-example-m3-java/README.md)); one report-execution defect found and fixed in the Node fixture (calibration item 1 in [validation-results-2026-08.md](docs/validation-results-2026-08.md)); Phase 3 exit criteria all satisfied; standard classified *validated, not production-proven*.

---

## Phase 2 — Rule Catalog (August 2026)

Authoring of the full Layer 2 catalog in seven batches, each with independent CAPire verification, candidate dispositions ([references/candidate-dispositions.md](references/candidate-dispositions.md)), and no severity/authority inflation:

- **Batch 1** — Security · Multitenancy (24 rules).
- **Batch 2** — Architecture · CDS · Services & APIs (26 rules).
- **Batch 3** — Data · Transactions · Events (23 rules).
- **Batch 4** — Business Logic · Integration (12 rules).
- **Batch 5** — Testing · Error handling · Observability (18 rules).
- **Batch 6** — Performance · Extensibility · Privacy (15 rules).
- **Batch 7** — Deployment · CI/CD · Versions · Operations · Dependencies (16 rules).

**Total: 134 rules.** No rule invented without SAP evidence (or explicit GEN/ORG/AI-REC labeling); every load-bearing claim verified against live CAPire before authoring; ~25 documented verification corrections during authoring (e.g., "Bad Practices" page removed → source map updated; hdbcds guidance clarified; 5xx sanitization runtime-scoped correctly).

---

## Phase 1 — Research & Foundation (August 2026)

- Six-agent parallel CAPire research fan-out establishing the [source map](references/sap-cap-sources.md), [candidate rules](references/candidate-rules.md), and [research gaps](references/research-gaps.md).
- Three-layer architecture ([standard-architecture.md](docs/standard-architecture.md)), authority taxonomy ([authority-levels.md](docs/authority-levels.md)), and version policy ([version-management.md](docs/version-management.md)).
- M0–M9 lifecycle ([lifecycle.md](development/lifecycle.md)), review and development models ([review-model.md](reviews/review-model.md), [development-model.md](development/development-model.md)).
- Layer-3 AI rule family ([standards/ai/](standards/ai/README.md)): AI-DEV, AI-REVIEW, AI-TEST, AI-DOC.

---

## Governance conventions applied throughout

- Rule IDs stable, never reused.
- SAP guidance never presented as organizational policy; ORG rules explicitly marked `ORG-PENDING`.
- CAPire evidence dated; version-sensitive rules flagged ⏱.
- Cross-references validated; no orphan rules; no dangling matrix rows.
- Historical pilot reports preserved as-authored (only one forward-only anonymization).
- No amendments to previous commits; no history rewrites.
