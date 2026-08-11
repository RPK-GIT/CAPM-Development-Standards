# Candidate Rule Dispositions

Record of what happened to each candidate from [candidate-rules.md](candidate-rules.md) as categories are formalized into the [Layer 2 catalog](../standards/rules/README.md). A candidate becomes normative only through this process. Disposition values: **ACCEPTED** (as inventoried) · **ACCEPTED WITH MODIFICATION** · **MERGED** (into another rule) · **DOWNGRADED** (severity/authority reduced) · **REJECTED** · **DEFERRED** (to a later batch or pending ORG decision).

## Batch 1 — Security & Multitenancy (2026-08-11)

Verification basis: all SAP-REQ/SAP-REC claims re-verified against live official documentation on 2026-08-11 (targeted verification pass; see notes column for the three claims that required correction). Final rules: [security.md](../standards/rules/security.md) (18), [multitenancy.md](../standards/rules/multitenancy.md) (6).

### Security candidates (18 reviewed)

| Candidate (candidate-rules.md §CAP-SEC) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Explicit authorization on every exposed service | ACCEPTED | CAP-SEC-001 | SAP-REQ re-verified ("By default, CDS services have no access control") |
| #2 No mocked/dummy auth in production | ACCEPTED | CAP-SEC-002 | Verbatim SAP warning confirmed |
| #3 Keep deny-by-default (`restrict_all_services`) | ACCEPTED | CAP-SEC-003 | Node.js-only; Java counterpart is CAP-SEC-004 |
| #4 Java strict authentication mode (`model-strict`) | ACCEPTED WITH MODIFICATION | CAP-SEC-004 | Reframed: rule forbids *weakening* documented defaults (SAP-REC); the *mandate* of `model-strict` remains an ORG decision — G-15 kept open, referenced from the rule |
| #5 Java: verify security deps AND binding present | ACCEPTED | CAP-SEC-005 | Fails-open behavior confirmed verbatim; added mandatory unauthenticated-rejection verification |
| #6 IAS-first for new projects | DOWNGRADED | CAP-SEC-006 | High → **Medium**: violating SAP's direction is strategic debt, not a vulnerability. New gap **G-43** added (XSUAA→IAS migration policy for existing apps) |
| #7 IAS tokens carry no scopes — pair with AMS | DOWNGRADED | CAP-SEC-007 | Critical → **High**: direct failure is a broken/improvised authorization layer; unauthorized access requires a secondary mistake. Reframed to "explicit authorization provisioning (AMS or documented alternative)" |
| #8 Business roles; exact scope↔role mapping | ACCEPTED | CAP-SEC-008 | "Exactly match … checked at runtime" confirmed |
| #9 Technical roles never in business roles | ACCEPTED | CAP-SEC-009 | Initial verification could not find the named roles (`mtcallback` etc.) on the checked pages; **direct re-verification found the exact statement on the current data-protection page** — SAP-REQ retained. Absorbs the technical-roles clause of MT candidate #4 |
| #10 Instance-based authorization declaratively | ACCEPTED | CAP-SEC-010 | — |
| #11 Composition/association exposure review | ACCEPTED | CAP-SEC-011 | Both runtime enforcement gaps confirmed verbatim (Java composition children; Node.js target-entity-only evaluation) |
| #12 Declarative validation on externally writable input | ACCEPTED WITH MODIFICATION | CAP-SEC-012 | Expanded to cover mass-assignment protection (`@readonly`/projection exclusion) and action/function parameters; future CAP-SRV validation candidate will cross-reference this rule instead of duplicating |
| #13 No user input in query structure | ACCEPTED WITH MODIFICATION | CAP-SEC-013 | Expanded into the single injection-safety rule, absorbing candidate **CAP-DB #6** (string concatenation, parenthesized tagged templates, `CQL.param`/`constant`, `hdb.` hints) |
| #14 DoS limits configured | ACCEPTED | CAP-SEC-014 | Rule enforces a *recorded decision*; concrete thresholds remain ORG (G-07 rate limits, G-08 Node `$expand` handler pattern). Java property name corrected to documented `cds.query.restrictions.expand.maxLevels` |
| #15 App Router is not a security boundary | ACCEPTED | CAP-SEC-015 | SAP wording confirmed incl. the mandated authentication tests |
| #16 Secure logging | ACCEPTED | CAP-SEC-016 | Authority SAP-REC; Java escaping and Node CRLF-safe API confirmed |
| #17 `@PersonalData` on all personal data | DEFERRED | — (future CAP-PRIV batch) | Identical scope to candidate CAP-PRIV #1; authored once in the data-privacy category to avoid duplication. Not dropped — deferred with this record |
| #18 No secrets on disk in development | ACCEPTED | CAP-SEC-017 | Extended with the consuming-services destination-credential warning (cross-reference target for future CAP-INT #4) |

### Cross-category candidates disposed in this batch

| Candidate | Disposition | Final rule | Notes |
|---|---|---|---|
| CAP-DB #6 Injection-safe query construction | MERGED | CAP-SEC-013 | Injection is fundamentally a security concern; the future CAP-DB category will cross-reference, not restate |
| CAP-OPS #3 MCP exposure governed | MERGED (and modified) | CAP-SEC-018 | Moved to Security per Phase 2 scope. **Modification from verification:** the claim "full CAP authN/authZ enforced on MCP requests" is *not* stated on the SAP MCP page — the rule now requires explicit verification by test instead of asserting SAP-guaranteed enforcement. Documented warnings (no prompt-injection validation/rate limiting/audit; "must not be used as a gateway or proxy for SAP Application APIs") confirmed verbatim |

### Multitenancy candidates (5 reviewed, 1 new rule authored)

| Candidate (candidate-rules.md §CAP-MT) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Streamlined MTX only | DOWNGRADED | CAP-MT-001 | Critical → **High**: legacy MTX still functions; the risk is an unsupported security-fix/upgrade path — gate-blocking, not immediate exposure |
| #2 MTX sidecar (Java mandatory / Node prod recommended) | ACCEPTED WITH MODIFICATION | CAP-MT-002 | Added the documented single-instance-first initial rollout (t0 conflict avoidance); split authority explicitly (SAP-REQ Java / SAP-REC Node) |
| #3 Strict per-tenant isolation | ACCEPTED | CAP-MT-003 | **Citation correction from verification:** Node.js mechanism wording sourced to the data-protection closure guidance + `cds.context` re-sourced to node.js/cds-tx (not the data-protection page) |
| #4 Idempotent subscribe handlers + technical roles | ACCEPTED WITH MODIFICATION | CAP-MT-004 | **Split:** technical-roles clause merged into CAP-SEC-009; remaining lifecycle-idempotency rule downgraded Critical → **High** (lifecycle integrity, not data exposure) and extended to unsubscribe/upgrade re-runnability |
| #5 Upgrade all tenants before serving | ACCEPTED | CAP-MT-005 | SAP ordering statement confirmed verbatim; orchestration specifics remain ORG (G-34) |
| — (scope item: background jobs & tenant context) | NEW RULE | CAP-MT-006 | Authored from newly verified evidence: Java `RequestContextRunner`/`TenantProviderService` (java/multitenancy, request-contexts) + cap-users "don't propagate named users to asynchronous requests" (SAP-REC). Not in the Phase 1 inventory — recorded here as batch addition |

### Batch summary

| Metric | Security | Multitenancy |
|---|---|---|
| Candidates reviewed | 18 (+2 cross-category) | 5 |
| Accepted unchanged | 12 | 2 |
| Accepted with modification | 3 | 2 |
| Merged (into batch rules) | 2 (CAP-DB #6, CAP-OPS #3) | — (1 clause merged out to CAP-SEC-009) |
| Downgraded | 2 | 1 |
| Rejected | 0 | 0 |
| Deferred | 1 (#17 → CAP-PRIV) | 0 |
| New rules from scope | 1 (CAP-SEC-018, via merge) | 1 (CAP-MT-006) |
| **Final rules** | **18** | **6** |

Severity distribution of the 24 rules: **9 Critical / 12 High / 3 Medium / 0 Low.** Every Critical rule carries an explicit "Critical justification" in its Rationale.

### Scope items reviewed with no rule created

- **Mass assignment / unintended property updates** — covered by CAP-SEC-012 (`@readonly`, projection exclusion) plus the future CAP-SRV projection-tailoring rule; no separate rule needed.
- **JWT/token handling mechanics** — CAP delegates token validation to `@sap/xssec`/CAP Java + platform; no application-level SAP guidance beyond CAP-SEC-002/-005/-007; nothing to formalize without inventing requirements.
- **Service-to-service authentication** — covered by CAP-SEC-009 (`internal-user` isolation) and CAP-SEC-017 (credentials); remote-service consumption security details belong to the future CAP-INT batch.
- **Security-sensitive error handling** — SAP's 5xx sanitization guidance is Node.js error-handling material (candidate CAP-ERR #2); deferred to the CAP-ERR batch to avoid category duplication.
- **Authorization audit logging** — SAP explicitly documents CAP does *not* log authorization decisions automatically; remains ORG gap G-06 (no rule inventable without an ORG pattern decision).
