# AI-TEST — Testing Rules for Claude Code

Authority: **ORG** (all rules). Runtime: Both. Status: Active.
These rules govern how Claude Code writes and uses tests. They complement the Layer 2 `CAP-TEST` rules (which define what the *application's* test suite must look like).

---

## AI-TEST-001 — Test behavior through service interfaces
**Severity: High.** Tests Claude writes MUST primarily exercise business behavior through CAP service interfaces (HTTP/OData via `cds.test` in Node.js; service/adapter layer via the CAP Java test support), not by unit-testing private helper internals. Framework behavior itself (generic CRUD, annotations) is not re-tested; *our* logic and configuration are.

## AI-TEST-002 — Use CAP-native test tooling
**Severity: High.** Node.js tests MUST use `cds.test` (with an in-memory database where applicable); CAP Java tests MUST use the documented Spring test support. Custom test harnesses that replicate what CAP test tooling provides require documented justification.

## AI-TEST-003 — New behavior ⇒ new test; changed behavior ⇒ changed test
**Severity: High.** For every functional change, Claude MUST identify the test that covers it. If none exists, one is written. If an existing test's expectation legitimately changes, the change is stated and justified in the completion report — see AI-DEV-014 for the prohibition on gutting tests.

## AI-TEST-004 — Cover the unhappy paths that matter
**Severity: High.** Tests MUST include: authorization denial for restricted operations, validation rejection for invalid input, and error behavior for the failure modes the code explicitly handles. A suite that only tests success paths does not satisfy AI-DEV-007.

## AI-TEST-005 — Deterministic and independent tests
**Severity: Medium.** Tests MUST NOT depend on execution order, wall-clock timing, shared mutable state across tests, or live external systems. External/remote services are mocked using CAP's documented mocking support; anything unmockable is an integration test, explicitly marked and separated.

## AI-TEST-006 — Verify by running, not by reading
**Severity: Critical.** Claude MUST actually execute the tests it writes or changes and report the real outcome. Reporting expected-but-unexecuted test results violates AI-DEV-016.

## AI-TEST-007 — Report coverage honestly
**Severity: Medium.** When reporting test status, Claude MUST state what is *not* covered (scenarios, components) rather than implying completeness. Coverage metrics, if quoted, come from an actual tool run.
