# M3 — Services / API

Operational checklist for the [M3 milestone](../lifecycle.md#m3--services--api). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md).

## Purpose
Use-case service facades with deliberate exposure, declared validation, and the authorization model — the API contract, decided here, security-verified at M6.

## Entry criteria
M2 PASS (domain model deployed).

## Applicable standards

**Primary (15):**

| Rule | Gate | Cond | Concern |
|---|---|---|---|
| CAP-SRV-001 | SOFT | — | Use-case projections, not persistence dumps |
| CAP-SRV-002 | SOFT | — | Generic providers relied on |
| CAP-SRV-003 | SOFT | — | Declarative before imperative |
| CAP-SRV-004 | SOFT | CUSTOM-OPS | Actions/functions semantics |
| CAP-SRV-005 | SOFT | ODATA | OData V4; V2 only justified |
| CAP-SRV-006 | SOFT | — | Explicit protocol exposure |
| CAP-SRV-007 | SOFT | DRAFTS | Framework draft handling |
| CAP-SRV-008 | SOFT | CONCURRENT-EDIT | `@odata.etag` concurrency decision |
| CAP-SRV-009 | SOFT | MEDIA | CAP media handling |
| CAP-SEC-001 | **HARD** | — | Explicit authorization per service |
| CAP-SEC-010 | SOFT | INSTANCE-AUTH | Declarative instance restrictions |
| CAP-SEC-011 | **HARD** | — | Association/composition exposure review |
| CAP-SEC-012 | SOFT | — | Declarative input validation |
| CAP-SEC-018 | **HARD** | MCP | MCP exposure governance |
| CAP-PERF-001 | SOFT | CONCURRENT-PAGING+ODATA | Reliable pagination |

**Supporting:** CAP-ARCH-003 (service cut holds), CAP-CDS-007 (annotation placement), CAP-SEC-014 (limits sketched), CAP-ERR-001/-005 (error contract: stable codes, targets).

## Required evidence
`srv/**/*.cds` service definitions as tailored projections; authorization aspect file (`@requires`/`@restrict` per service — deliberate `@requires:'any'` documented); validation annotations on writable elements; protocol annotations (`@protocol:'none'` for internal; Java V2 stance); exposure-review notes for association paths (SEC-011); API documentation.

## Required tests
API tests for generic CRUD exposure and validation rejection (`cds.test`/MockMvc) — allow/deny authorization tests may start here (completed by M6/M7).

## Review procedure
1. Enumerate services and exposed entities; compare against the domain model (SRV-001's 1:1 detection).
2. **HARD gates:** SEC-001 — every non-internal service carries explicit authorization (fallbacks don't count); SEC-011 — walk every exposed association/composition, verify mitigation per path; SEC-018 — any `@mcp` service has authorization + no SAP-API proxying (if MCP flag set).
3. Evaluate declarative coverage: validations as annotations (SEC-012/SRV-003), restrictions as `@readonly`/`@insertonly`.
4. Check protocol deliberateness (SRV-005/-006 — Java's default V2 exposure decided).
5. Conditional rules per profile flags (drafts, media, concurrency, paging).

## Remediation expectations
Authorization/exposure HARD findings block — fixing the service cut is still cheap here. Validation gaps close before M4 writes handler logic that assumes them.

## Exit criteria
API review PASS; API tests green; exposure and authorization models recorded; HARD gates clear. Result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results).
