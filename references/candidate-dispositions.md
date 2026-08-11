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

### Scope items reviewed with no rule created (Batch 1)

- **Mass assignment / unintended property updates** — covered by CAP-SEC-012 (`@readonly`, projection exclusion) plus the future CAP-SRV projection-tailoring rule; no separate rule needed.
- **JWT/token handling mechanics** — CAP delegates token validation to `@sap/xssec`/CAP Java + platform; no application-level SAP guidance beyond CAP-SEC-002/-005/-007; nothing to formalize without inventing requirements.
- **Service-to-service authentication** — covered by CAP-SEC-009 (`internal-user` isolation) and CAP-SEC-017 (credentials); remote-service consumption security details belong to the future CAP-INT batch.
- **Security-sensitive error handling** — SAP's 5xx sanitization guidance is Node.js error-handling material (candidate CAP-ERR #2); deferred to the CAP-ERR batch to avoid category duplication.
- **Authorization audit logging** — SAP explicitly documents CAP does *not* log authorization decisions automatically; remains ORG gap G-06 (no rule inventable without an ORG pattern decision).

---

## Batch 2 — Architecture, CDS & Services (2026-08-11)

> **Baseline note:** the Phase 2 planning brief referred to a completed "Data, Transactions & Events" batch. **No such batch exists in the repository** (no commits beyond Batch 1; `data-persistence.md`, `transactions.md`, `events-messaging.md` are unauthored stubs). Those categories remain candidate inventory, and cross-references below treat them as *future* categories. This is therefore the second authored batch.

Verification basis: two targeted verification passes (architecture/CDS pages; services pages) against live official documentation on 2026-08-11. Material corrections from verification are marked ⚠ in the notes. Final rules: [architecture.md](../standards/rules/architecture.md) (7), [cds-modeling.md](../standards/rules/cds-modeling.md) (11), [services-apis.md](../standards/rules/services-apis.md) (9).

### Architecture candidates (8 reviewed, 1 new rule)

| Candidate (candidate-rules.md §CAP-ARCH) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Standard project layout, convention over configuration | ACCEPTED WITH MODIFICATION | CAP-ARCH-001 | Absorbs candidate #8 (CSV naming) and the Node.js impl-file convention; severity High → **Medium** (tooling/maintainability, not integrity) |
| #2 No abstraction layers on top of CAP | ACCEPTED WITH MODIFICATION | CAP-ARCH-002 | ⚠ SAP's *Bad Practices* page is **removed** (404, no successor; whole `/docs/about/` section gone). Rule re-scoped to what the live concepts page documents (DAO/DTO/Repository/Active-Records criticism; determinations in event handlers). Severity Critical → **High** (worst *architectural* failure, but not direct data/security exposure) |
| #3 Single-purpose services per use case | DOWNGRADED | CAP-ARCH-003 | Critical → **High**: design defect that complicates security/API surface; direct exposure is owned by CAP-SEC-001/CAP-SRV-001 |
| #4 Services expose tailored projections | MERGED | → CAP-SRV-001 | Relocated to the Services category where projection design lives |
| #5 Stateless services, passive data | ACCEPTED | CAP-ARCH-004 | High kept: scale-out reliability; escalates to CAP-MT-003 (Critical) in multitenant contexts |
| #6 Platform/protocol-agnostic code | DOWNGRADED | CAP-ARCH-005 | High → **Medium**: capability erosion/maintainability; integrity/security consequences owned by cross-referenced rules |
| #7 Modulith-first, late-cut microservices | ACCEPTED | CAP-ARCH-006 | Medium kept; ⚠ verification found SAP's wording *stronger* than inventoried ("Avoid premature cuts… Go for a modulith… only when you really need to") |
| #8 CSV seed-data naming convention | MERGED | → CAP-ARCH-001 | Framework-convention concern, same rule |
| — (batch scope: ADR-worthy decisions) | NEW RULE | CAP-ARCH-007 | **ORG** authority (SAP prescribes no ADRs); scoped to material decisions only; the record other rules' exceptions key on (AI-DOC-001/-002 alignment) |

### CDS candidates (13 reviewed)

| Candidate (§CAP-CDS) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Naming casing | ACCEPTED WITH MODIFICATION | CAP-CDS-001 | Absorbs #2 and #3 into one naming rule; severity Medium → **Low** (style). Org vocabulary beyond SAP's checklist stays gap G-19 |
| #2 Plural entities, singular types | MERGED | → CAP-CDS-001 | — |
| #3 Concise names, `ID` keys | MERGED | → CAP-CDS-001 | — |
| #4 Canonical UUID keys via `cuid` | DOWNGRADED | CAP-CDS-002 | High → **Medium** ("prefer" wording; integration friction, not breakage); semantic keys explicitly carved in |
| #5 UUIDs are opaque | DOWNGRADED | CAP-CDS-003 | ⚠ Authority SAP-REQ → **SAP-REC** per verified wording ("anti pattern", Don't/Avoid — no must/never); severity High → **Medium** |
| #6 Use `@sap/cds/common` | DOWNGRADED | CAP-CDS-004 | High → **Medium**; ⚠ the "managed fields are write-protected" claim is **not on the cited page** — rationale rebuilt on the documented benefits (proven practices, interoperability, clean models) |
| #7 Prefer managed associations | DOWNGRADED | CAP-CDS-005 | High → **Medium** (comprehensibility; unmanaged modeling still functions) |
| #8 Compositions for contained-in; no composition-of-one | ACCEPTED | CAP-CDS-006 | **High kept** with explicit justification: containment choice decides cascade/deep-op/draft lifecycle → data-integrity defects; composition-of-one discouragement verified verbatim with SAP's reasons |
| #9 Aspects for separation of concerns | DOWNGRADED | CAP-CDS-007 | High → **Medium** (readability/coupling; compiler merges either way) |
| #10 `localized` keyword | DOWNGRADED | CAP-CDS-008 | High → **Medium**; restrictions (keys not associations; sub-elements ignored) verified and embedded |
| #11 Temporal via aspect; writes in custom handlers | ACCEPTED WITH MODIFICATION | CAP-CDS-009 | "Writing temporal data must be done in custom handlers" verified verbatim (SAP-REQ on the write path); added the un-annotated-validity-columns failure case |
| #12 Flat models; reuse-justified types | DOWNGRADED | CAP-CDS-010 | Medium → **Low** (style with documented grounding) |
| #13 Stable namespaces | ACCEPTED | CAP-CDS-011 | Medium kept — rename cost bleeds into persistence/CSV/API names |

### Services candidates (10 reviewed + 1 cross-category pull)

| Candidate (§CAP-SRV) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Prefer generic providers | ACCEPTED | CAP-SRV-002 | High kept: reimplementation loses framework-enforced behavior (integrity/security adjacency) |
| #2 Declarative input validation first | MERGED | → CAP-SEC-012 | Batch 1 decision now executed: single validation rule, cross-referenced from CAP-SRV-003 — not re-authored |
| #3 Declarative restrictions (`@readonly`/`@insertonly`) | MERGED | → CAP-SRV-003 | Folded as declarative means within the declarative-first rule |
| #4 Actions vs functions semantics | ACCEPTED | CAP-SRV-004 | SAP-REQ ("Actions modify data in the server. Functions retrieve data."); extended to prohibit CRUD-smuggled operations and framework-bypassing routes |
| #5 OData V4 default; V2 deprecated | ACCEPTED | CAP-SRV-005 | Deprecation quote verified with its two sanctioned exceptions; per-service justification requirement added |
| #6 Explicit protocol exposure | ACCEPTED WITH MODIFICATION | CAP-SRV-006 | ⚠ Verification confirmed Java serves **odata-v4 AND odata-v2 by default** — the rule now requires an explicit V2-exposure decision in Java projects |
| #7 Framework draft handling | DOWNGRADED | CAP-SRV-007 | High → **Medium** (framework duplication; the security-relevant validation caveat lives in CAP-SEC-012) |
| #8 Validate active data, not only drafts | MERGED | → CAP-SEC-012 | Already embedded there in Batch 1 (fiori caveat verified this batch, verbatim) |
| #9 Optimistic concurrency via ETags | ACCEPTED WITH MODIFICATION | CAP-SRV-008 | ⚠ Cite **412 only** (409 not on the page); Java `rowCount()==0` behavior included; "recorded last-write-wins decision" added as the compliant alternative |
| #10 Media via `@Core.MediaType` + streaming | DOWNGRADED | CAP-SRV-009 | ⚠ No "strongly recommended" wording on the current page → authority stays SAP-REC but rationale rebuilt on the documented mechanism ("streamed automatically"); severity High → **Medium**; Java stream-closing caveat dropped (not on page) |
| CAP-LOGIC #1 Declarative before imperative (cross-category) | MERGED | → CAP-SRV-003 | The "Consider first… declarative techniques" guidance verified verbatim; imperative logic explicitly legitimized for genuine domain behavior |

### Batch summary

| Metric | Architecture | CDS | Services |
|---|---|---|---|
| Candidates reviewed | 8 (+1 new) | 13 | 10 (+1 cross-category) |
| Accepted unchanged | 2 | 2 | 3 |
| Accepted with modification | 2 | 2 | 2 |
| Merged | 2 | 2 | 3 (incl. CAP-LOGIC #1) |
| Downgraded | 2 | 7 | 2 |
| Rejected | 0 | 0 | 0 |
| Deferred | 0 | 0 | 0 |
| New rules | 1 (CAP-ARCH-007, ORG) | 0 | 0 |
| **Final rules** | **7** | **11** | **9** |

Batch severity: **0 Critical / 7 High / 18 Medium / 2 Low** — consistent with the batch guidance that architecture/modeling violations are predominantly quality debt, with High reserved for rules whose violation materially affects integrity, exposure surface, API compatibility, or scale-out reliability. Authority: 3 SAP-REQ (CAP-CDS-009 write path, CAP-SRV-004, CAP-SRV-005), 23 SAP-REC, 1 ORG (CAP-ARCH-007). No runtime-specific rules; runtime-specific *content* is flagged inside CAP-ARCH-001 (Node impl-file convention), CAP-SRV-004 (Java `@On` requirement), CAP-SRV-005/-006 (Java default V2 exposure), CAP-SRV-008 (Java `rowCount()`).

### Scope items reviewed with no rule created (Batch 2)

- **DDD terminology (bounded contexts, aggregates, repositories)** — deliberately NOT imposed: SAP's live docs criticize repository/Active-Record patterns and never mandate DDD vocabulary; CAP's own concepts (use-case services, compositions as document trees) carry the intent. Creating DDD-named rules would manufacture authority.
- **Layer-count / application-vs-domain-layer prescriptions** — no SAP evidence for any specific layering beyond db/srv/app and hexagonal-by-construction; rejected as org design preference not worth an ORG rule.
- **API versioning / breaking-change management** — SAP is silent; remains gap G-22 (no rule inventable).
- **Multi-protocol exposure policy (same service on OData+REST+GraphQL)** — mechanism documented, policy absent; remains gap G-23, partially mitigated by CAP-SRV-006's explicitness requirement.
- **Pagination limit values** — mechanism confirmed (default 1,000 truncation) but rule ownership stays with CAP-SEC-014 (decision) and future CAP-PERF (values); not duplicated here.
- **Draft-enablement criteria (when to enable drafts)** — SAP documents the mechanism, not criteria; remains gap G-20; CAP-SRV-007 governs *how*, not *whether*.
- **Bad-practices content (code generation, "abstracting from an abstraction")** — page removed from official docs; the portions without live-page support were narrowed out of CAP-ARCH-002 rather than cited to a dead URL; source map updated to track the removal.
