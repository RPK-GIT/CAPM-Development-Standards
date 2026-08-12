# M9 — Production Readiness (Go-Live Gate)

Operational checklist for the [M9 milestone](../lifecycle.md#m9--production-readiness). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md). This milestone **is** CAP-OPS-003's assessment — the record produced here is the go-live evidence.

## Purpose
The final, aggregating assessment: full standard review + operational readiness demonstrated — not re-reviewing everything from scratch, but re-verifying what production depends on and proving operations works.

## Entry criteria
M8 PASS (staging deployment from the pipeline).

## Applicable standards

**Primary (5):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-OPS-002 | SOFT | UI | Production UI entry point serving |
| CAP-OPS-003 | **HARD** *(ORG-PENDING)* | — | The completed readiness assessment itself |
| CAP-LOG-003 | SOFT | — | Correlation demonstrated end-to-end |
| CAP-LOG-004 | SOFT | TELEMETRY | SAP OTel tooling (if adopted) |
| CAP-PRIV-004 | SOFT | PERSONAL-DATA | Documented retention/erasure approach |

**FINAL-GATE re-verifications (the production-critical set):** CAP-SEC-001, CAP-SEC-002, CAP-SEC-009, CAP-SEC-015 · CAP-MT-003, CAP-MT-005 (MT) · CAP-EVT-002, CAP-EVT-003, CAP-EVT-005 (EVENTING) · CAP-PRIV-001, CAP-PRIV-002 (PERSONAL-DATA) · CAP-VER-002 · CAP-TEST-007 · CAP-LOG-002, CAP-LOG-005 (JAVA) · CAP-OPS-001 · CAP-CICD-003 *(ORG-PENDING)* · CAP-ARCH-007 *(ORG-PENDING —* every deviation across M1–M8 has its record).

**Plus:** the full-catalog review required by CAP-OPS-003 (scoped by the capability profile — this is the one point where the whole applicable catalog is swept; unresolved **Medium** findings from earlier milestones are dispositioned here per the lifecycle: resolved or formally accepted).

## Required evidence
- Completed [milestone checklist](../../templates/milestone-checklist-template.md) + full standard review report ([review model](../../reviews/review-model.md)).
- **Operational drill results:** one request traced end-to-end by correlation ID through structured production-config logs (LOG-002/-003); health probes answered live (OPS-001); UI entry URL serving (OPS-002); where alerting exists per ORG policy (G-37), a test alert fired.
- Operations runbook: dead-letter procedure (EVT-005), rollback (CICD-002 scope), tenant-upgrade sequence (MT-005), upgrade posture (VER-002), on-call basics.
- Audit-logging verification in production configuration (PRIV-002); retention/erasure document (PRIV-004).
- Exception register: every AI-DOC-002 record referenced from this assessment; every ORG-PENDING finding explicitly listed.
- Performance validation against M0 NFRs (values are ORG G-04 — evidence of the validation run is the check).

## Required tests
Post-deploy smoke suite green in the production-like environment; load/perf validation vs NFRs; restart/failover behavior observed.

## Review procedure
1. Verify M0–M8 gate results are on record (any skipped gate → NOT READY — gates may not be back-filled).
2. Run the FINAL-GATE list above against the production configuration/deployment (profile-filtered).
3. Execute/verify the operational drill; attach outputs as DIRECT evidence.
4. Sweep the remaining applicable catalog per CAP-OPS-003 (report-by-exception: PASS lines may be summarized; every FAIL/NOT-ASSESSABLE itemized).
5. Disposition all open Medium findings (resolve or formally accept); confirm zero uncovered Critical/High.
6. Produce the assessment record; the milestone result is the go-live decision input.

## Remediation expectations
Any uncovered HARD violation → FAIL, no go-live. NOT READY (missing drill evidence, missing runbook) is not negotiable into PASS WITH EXCEPTIONS — exceptions cover *rule* deviations, not missing evidence.

## Exit criteria
Assessment record complete; zero uncovered Critical/High; Mediums dispositioned; drill evidence attached; go-live approval recorded. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results) — re-assessment required on major change (new tenancy model, deployment target, CAP major) per the lifecycle.
