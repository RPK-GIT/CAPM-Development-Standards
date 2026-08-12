# M5 — Integration & Events

Operational checklist for the [M5 milestone](../lifecycle.md#m5--integration). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md). **Entirely conditional:** projects without REMOTE/EVENTING/EXTENSIBILITY flags mark this milestone NOT APPLICABLE (with the M0 profile as evidence).

## Purpose
Remote consumption through CAP's abstractions, reliable eventing on the transactional queue, and — for SaaS providers — the extension offering.

## Entry criteria
M4 PASS; M0 profile flags REMOTE and/or EVENTING and/or EXTENSIBILITY set.

## Applicable standards

**Primary (16):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-INT-001 | **HARD** | REMOTE | `cds import`; never copy CDS; V4 between CAP services |
| CAP-INT-002 | SOFT | REMOTE | CAP remote services + destinations |
| CAP-INT-003 | SOFT | REMOTE | Consumption views |
| CAP-INT-004 | SOFT | REMOTE | Remote mocks for dev/tests |
| CAP-INT-005 | SOFT | MASHUP | Bounded cross-source expands |
| CAP-INT-006 | SOFT | REMOTE+MT | Destination retrieval strategy |
| CAP-INT-007 | SOFT | REMOTE | Remote-failure design |
| CAP-EVT-001 | SOFT | EVENTING | Protocol-agnostic events |
| CAP-EVT-002 | **HARD** | EVENTING | Transactional queue on |
| CAP-EVT-003 | **HARD** | EVENTING | Idempotent handlers |
| CAP-EVT-004 | **HARD** | EVENTING | Privileged processing; claims in payload; no secrets in headers |
| CAP-EVT-005 | SOFT | EVENTING | Dead-letter operations |
| CAP-EVT-007 | ADV | EVENTING | Local messaging in dev |
| CAP-EXT-002 | **HARD** | EXTENSIBILITY | Configured extension allowlist |
| CAP-EXT-003 | SOFT | EXTENSIBILITY | Supported extension workflow |
| CAP-EXT-004 | SOFT | FEATURE-TOGGLES | Feature isolation; schema uniformity |

**Supporting:** CAP-EVT-006 (broker decided at M1 — wiring verified here), CAP-TXN-005 (remote calls in flows: partial-failure design), CAP-MT-004/-006 (lifecycle handlers, tenant context in processing), CAP-SEC-017 (no destination credentials), CAP-CDS-003 (foreign keys accepted opaquely), CAP-LOG-003 (correlation across custom clients).

## Required evidence
`srv/external/` imported definitions + `cds.requires` configs with destinations; consumption views; mock data (`srv/external/data/*.csv`, Java `--with-mocks`); event declarations + emit/consume sites (all awaited); idempotency mechanisms per handler (file:line); no queue-disabling production config; `cds.unqueued`/immediate-dispatch justifications; dead-letter service/runbook section; extension allowlist config + workflow docs (if EXTENSIBILITY); the app running fully locally on mocks.

## Required tests
Integration tests against mocks incl. remote-failure paths; duplicate-delivery tests for critical event handlers (EVT-003 evidence, completed at M7); local run without external connectivity.

## Review procedure
1. Apply the profile filter; mark absent capabilities NOT APPLICABLE.
2. **HARD gates:** INT-001 (no copied CDS/shadow tables; V4 CAP-to-CAP), EVT-002 (queue defaults intact in production config), EVT-003 (idempotency per consumer), EVT-004 (no role checks in async handlers; no sensitive headers), EXT-002 (allowlist configured, `x_` prefix).
3. Remote-consumption pass: abstraction usage (INT-002 — hand-rolled clients?), views (INT-003), mocks (INT-004), fan-out bounds (INT-005), failure design (INT-007 — bounded waits, defined outcomes, no blind retries).
4. MT cross-checks where flagged: destination strategy (INT-006), lifecycle-handler idempotency (MT-004 — findings owned at M6).

## Remediation expectations
HARD findings block — queue/idempotency defects compound silently in production. Failure-design gaps (INT-007) may proceed with recorded justification only where the integration is non-critical.

## Exit criteria
Integration review PASS; app runs locally on mocks; HARD gates clear. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).
