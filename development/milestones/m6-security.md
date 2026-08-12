# M6 — Security

Operational checklist for the [M6 milestone](../lifecycle.md#m6--security). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md). The security gate: 14 primary rules **plus 8 FINAL-GATE re-verifications** of security rules whose subjects were built in M3–M5.

## Purpose
The dedicated security verification: production identity, authorization completeness, tenant isolation, secrets, audit logging — evaluated against the implementation as it now exists, not as designed.

## Entry criteria
M3–M5 PASS (services, logic, integration implemented).

## Applicable standards

**Primary (14):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-SEC-002 | **HARD** | — | Real identity service; no mock auth in production config |
| CAP-SEC-003 | **HARD** | NODE | `restrict_all_services` untouched |
| CAP-SEC-004 | **HARD** | JAVA | Java auth defaults not weakened |
| CAP-SEC-005 | **HARD** | JAVA | Security deps + binding both present |
| CAP-SEC-007 | **HARD** | IAS | AMS/authorization provisioning |
| CAP-SEC-008 | **HARD** | XSUAA | Scope↔role integrity (regenerated) |
| CAP-SEC-009 | **HARD** | — | Technical roles never business-assigned |
| CAP-SEC-014 | SOFT | — | DoS limits decided ($batch/$expand/pagination/rate) |
| CAP-SEC-015 | **HARD** | — | Backend auth independent of App Router (+ test) |
| CAP-SEC-016 | SOFT | — | Secure logging (also FINAL-GATE of M4 work) |
| CAP-SEC-017 | **HARD** | — | No secrets in repo/dev artifacts |
| CAP-MT-003 | **HARD** | MT | Strict tenant isolation (code scan + isolation tests) |
| CAP-MT-004 | SOFT | MT | Idempotent lifecycle handlers |
| CAP-PRIV-002 | **HARD** | PERSONAL-DATA | Audit logging wired for production |
| CAP-PRIV-003 | **HARD** | PDM | PDM service protected + flat |

**FINAL-GATE re-verifications (built earlier, re-checked now):** CAP-SEC-001, CAP-SEC-010, CAP-SEC-011, CAP-SEC-012, CAP-SEC-013, CAP-SEC-018, CAP-ERR-002, CAP-ERR-006 — implementation changes since M3/M4 can have invalidated them; re-run their detection on the current tree.

**Supporting:** CAP-EVT-004 (async privileged-context checks), CAP-TEST-005/-007 (mock users + the authorization test matrix — completed at M7, started here), CAP-PRIV-004 (retention approach drafted), CAP-EXT-002 (allowlist as security artifact).

## Required evidence
Production auth configuration per profile; `xs-security.json`/AMS artifacts diffed against model roles; role-collection definitions (or cockpit export); secret scans + `.gitignore` coverage; tenant-isolation test results (mock tenants t1/t2); audit-log production binding; authorization test evidence (allow + deny per restricted resource — at least the critical paths); security notes for every deviation (AI-DEV-013).

## Required tests
Authorization matrix tests (denial paths mandatory); unauthenticated-rejection test (suite or pipeline — its deployed execution is re-verified at M8); tenant-isolation tests (MT); audit-event smoke (modify annotated field → event) where personal data is processed.

## Review procedure
1. Run all 14 primary rules' detection against current code/config (filtered by NODE/JAVA/IAS/XSUAA/MT/PERSONAL-DATA/PDM flags).
2. Re-run the 8 FINAL-GATE rules' detection — treat any regression since M3/M4 as a new finding, not a footnote.
3. Verify test evidence exists for SEC-001/-015/MT-003 (the tests themselves are M7's completeness subject).
4. Aggregate: any uncovered HARD violation → FAIL; exceptions only via AI-DOC-002 with security-competent approver.

## Remediation expectations
Zero tolerance on HARD security findings — the lifecycle requires this gate to close with **zero Critical/High security findings** (or formally approved exceptions). Re-review of failed rules after remediation is mandatory (AI-REVIEW-012 separation: fix, then re-review).

## Exit criteria
Security review PASS with zero unresolved Critical/High findings; authorization test suite green; deviations documented. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).
