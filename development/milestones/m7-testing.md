# M7 — Testing & Quality Consolidation

Operational checklist for the [M7 milestone](../lifecycle.md#m7--testing-consolidation). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md).

## Purpose
Consolidate the suite built incrementally through M2–M6 into complete, deterministic, CI-executable evidence: behavior coverage, security matrix, parity-sensitive hybrid coverage, honest gap statement.

## Entry criteria
M4–M6 PASS (behavior implemented and security-gated; tests have accompanied code per AI-DEV-007 throughout).

## Applicable standards

**Primary (7):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-TEST-001 | SOFT | NODE | `cds.test` bootstrap + isolation |
| CAP-TEST-002 | SOFT | JAVA | Documented test layers; H2 ≤ 2.3.x; MTX → hybrid |
| CAP-TEST-003 | SOFT | NODE | Stable-contract assertions |
| CAP-TEST-004 | ADV | NODE | Runner portability |
| CAP-TEST-005 | SOFT | — | Mock users for authenticated flows |
| CAP-TEST-006 | SOFT | CLOUD-ONLY | Hybrid coverage for parity-sensitive features |
| CAP-TEST-007 | **HARD** | — | Security behavior verified (matrix + unauthenticated rejection) |

**Supporting (verification duties from earlier milestones consolidated here):** CAP-EVT-003 (duplicate-delivery tests), CAP-TXN-006 (no-floating-promises lint as preventive control), CAP-DB-002/-008 (hybrid tests where SQLite/H2 parity fails), CAP-INT-004 (mock-based integration tests in CI), CAP-CICD-001 (suite runs in the pipeline — configured at M8).

**Layer 3:** the AI-TEST family governs how tests are written; this milestone verifies what exists regardless of author.

## Required evidence
Full suite green in one command (and in CI); coverage measured with **honest gap statement** (AI-TEST-007 — no ORG threshold exists, G-01: presence and honesty are the check, not a percentage); the security test matrix mapped to restricted resources (TEST-007's per-resource allow+deny inventory); hybrid stage definition + last run for CLOUD-ONLY features; determinism evidence (no order dependence — shuffled run or static analysis per TEST-001).

## Required tests
The suite **is** the deliverable: behavior + unhappy paths per acceptance criterion; the authorization matrix; validation rejections; duplicate-delivery (EVENTING); isolation (MT — owned by MT-003, presence checked here); 412 conflict paths (CONCURRENT-EDIT).

## Review procedure
1. Execute the suite (or verify the CI run) — red suite → NOT READY, not FAIL-with-justification.
2. Evaluate TEST-001..006 per detection (filtered by runtime/CLOUD-ONLY).
3. **HARD gate TEST-007:** build the restricted-resource inventory from the M3/M6 review outputs; map deny tests per resource; any restricted resource without a denial test → FAIL.
4. Brittleness scan (TEST-003): message-text/snapshot/volatile-field assertions.
5. Verify the coverage gap statement exists and matches reality (spot-check two claimed-covered behaviors).

## Remediation expectations
TEST-007 gaps block. Flaky tests are fixed or quarantined-with-issue (never deleted to go green — AI-DEV-014 discipline applies to humans too via this gate).

## Exit criteria
Full suite green in CI; security matrix complete; known gaps documented and accepted; HARD gate clear. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).
