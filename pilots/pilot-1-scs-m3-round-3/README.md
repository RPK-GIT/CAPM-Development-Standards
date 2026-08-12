# Phase 4 — Pilot 1 · Round 3 · Controlled Real-Project Remediation Experiment (scs)

Round 3 closes the closed-loop objective Round 2 left half-open. **Pilot review against the applicable CAPM Development Standards** — NOT a certification.

| Field | Value |
|---|---|
| Target | `scs` (Supply Coverage Simplified) — same real project as Rounds 1 & 2 |
| Milestone | M3, RETROSPECTIVE mode, `project.milestone: LIVE` |
| Standard version | `99332a6` (post-Round-2) |
| Review date | 2026-08-12 |
| Remediation scope | **Root A (CAP-SEC-001, Critical HARD)** + **Root B partial (CAP-SEC-010, High SOFT)** via a new declarative-authorization aspect file at `srv/access-control.cds` in scs (a single new file; no other application code touched) |
| Result | 2 real FAIL → PASS (SEC-001, SEC-010); 9 unremediated FAILs correctly remain FAIL; **HARD gate CLEARED** — milestone is no longer at a HARD-blocker FAIL, but not-yet-PASS while SOFT items remain unjustified |
| Pilot classification | **GREEN** — every closed-loop criterion in the Round 3 brief was met |

## Contents

- [review-report.md](review-report.md) — the full Round 3 report incl. closed-loop validation, remaining-findings evidence, root-finding recap, and the honest not-yet-PASS gate state.

## Application-code artifact

The remediation itself lives in the `scs` project at `srv/access-control.cds` (49 lines, model-level annotations only). That file is **not copied here** — application code stays in the application repository. The report cites file:line evidence and the effective annotations from a compile-time inspection of the model.
