# M4 — Business Logic

Operational checklist for the [M4 milestone](../lifecycle.md#m4--business-logic). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md). The largest primary rule set (29) — filtered hard by runtime and capability flags.

## Purpose
Custom handlers implementing what generic providers don't: correct phases, transactional discipline, injection-safe data access, honest error behavior.

## Entry criteria
M3 PASS (services defined, validation/authorization declared).

## Applicable standards

**Primary (29)** — grouped; gate/condition per [matrix](../rule-milestone-matrix.md):

| Group | Rules (gate; condition) |
|---|---|
| Handler design | LOGIC-001 (S), LOGIC-002 (S), LOGIC-003 (S; NODE), LOGIC-004 (S; JAVA), LOGIC-005 (S; JAVA) |
| Data access | DB-004 (S), DB-005 (S; NODE), DB-006 (S; NODE+LOCALIZED), DB-008 (S; PESSIMISTIC-LOCKING), DB-009 (S; NODE), DB-010 (A; JAVA) |
| Transactions | TXN-001 (S), TXN-002 (S; NODE), TXN-003 (A; NODE), TXN-004 (S; JAVA), TXN-005 (S), **TXN-006 (H; NODE)** |
| Security in code | **SEC-013 (H)** injection safety, SEC-016 (S) log content |
| Errors | ERR-001 (S), **ERR-002 (H; NODE)** sanitization, ERR-003 (S), ERR-004 (S; NODE+ERROR-HOOKS), ERR-005 (S; UI), **ERR-006 (H)** message input validation |
| Async/tenancy | MT-006 (S; MT) background tenant context |
| Performance | PERF-005 (S; NODE+MASS-DATA), PERF-007 (S; GEN) no N+1 |
| Observability | LOG-001 (S) framework logging |

**Supporting:** ARCH-002/-004 (no layers/state creeping into code), SRV-002/-003/-004 (generic-provider and declarative discipline in implementations), SEC-010/-012 (handler queries preserve restrictions; imperative validation only where declarative can't), CDS-009 (temporal write handlers), PERF-003 (no filtering on live-calculated), TEST-001 (tests accompany code — AI-DEV-007).

## Required evidence
Handler implementations (`srv/**/*.js|ts`, `srv/src/main/java/**`) passing the per-rule scans (string-built queries, floating promises, phase misuse, swallowed errors, module-level state, N+1 loops — each with file:line results); behavior tests through service interfaces incl. unhappy paths; completion reports per increment (AI-DEV-008 self-validation tables).

## Required tests
Behavior tests per acceptance criterion incl. validation-rejection and error paths (AI-TEST-004); duplicate/rollback scenarios where handlers have side effects.

## Review procedure
1. Filter the primary set by runtime + profile flags (a Node single-tenant project drops LOGIC-004/-005, TXN-004, DB-010, MT-006 …).
2. **HARD gates:** SEC-013 (full injection scan per its detection — taint from `req.data` into query text/structure/hints), TXN-006 (floating-promise scan; `await srv.emit`), ERR-002 (`$sanitize`, error-handler leakage), ERR-006 (user input in messages).
3. Handler-design pass: phases (LOGIC-002 incl. Node concurrency + event-vs-request `on` semantics), placement (LOGIC-001/-005), registration correctness (LOGIC-003/-004).
4. Transaction pass: manual tx in handlers (TXN-001), unmanaged-context usage (TXN-002/-004), partial-failure design for multi-transaction flows (TXN-005).
5. Data-access pass: CQL-first (DB-004), runtime-specific constraints (DB-005/-006/-009), set-based access (PERF-007).
6. Error/logging pass: CAP error APIs (ERR-001), no swallowed errors (ERR-003), `cds.log`/SLF4J (LOG-001), no secrets/PII in logs (SEC-016).

## Remediation expectations
HARD findings block. Soft handler-quality findings are fixed in the same increment wherever the handler is still being touched — deferral to "cleanup later" requires the checklist justification.

## Exit criteria
All in-scope acceptance criteria demonstrably tested; scans clean or excepted; logic review PASS. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).
