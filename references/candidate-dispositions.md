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

---

## Batch 3 — Data, Transactions & Events (2026-08-11)

Verification basis: two targeted verification passes (databases/persistence pages; transactions/events pages) against live official documentation on 2026-08-11. Six documentation-precision corrections applied (marked ⚠). Final rules: [data-persistence.md](../standards/rules/data-persistence.md) (10), [transactions.md](../standards/rules/transactions.md) (6), [events-messaging.md](../standards/rules/events-messaging.md) (7).

### Data / persistence candidates (10 reviewed — #6 already dispositioned in Batch 1 — plus 1 cross-category pull, 1 new rule)

| Candidate (candidate-rules.md §CAP-DB) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 HANA Cloud for production | ACCEPTED | CAP-DB-001 | High kept; PostgreSQL "aren't **yet** supported" nuance preserved; QRC validation scope embedded |
| #2 No SQLite/H2 in production | ACCEPTED WITH MODIFICATION | CAP-DB-002 | "Not fit for production" verified verbatim (SAP-REQ); extended with dev/prod parity-limit awareness (locking, streaming, time-travel) and the documented in-memory-cache carve-out |
| #3 New `@cap-js` database services | ACCEPTED WITH MODIFICATION | CAP-DB-003 | Absorbs #4; ⚠ authority on migration corrected REQ → **SAP-REC** ("highly recommended", no removal date documented); the no-direct-drivers clause carries the normative "should not" wording (cite new-dbs, not the sqlite page) |
| #4 No direct driver dependencies | MERGED | → CAP-DB-003 | Same concern, same mechanism |
| #5 CQL over raw SQL | ACCEPTED | CAP-DB-004 | High kept; injection mechanics remain CAP-SEC-013 (cross-referenced, not restated); org criteria for native SQL stay gap G-24 |
| #6 Injection-safe query construction | (Batch 1) MERGED | → CAP-SEC-013 | Historical — recorded in Batch 1 |
| #7 Path expressions, not CQN UNION/JOIN | DOWNGRADED | CAP-DB-005 | High → **Medium**: dropped-capability violations fail loudly at runtime (defects, not silent corruption); "dropped support" verified verbatim; virtual-elements-ignored behavior folded in |
| #8 `SELECT.localized` for localized reads | ACCEPTED | CAP-DB-006 | Medium kept — the silent wrong-language failure mode is the review value |
| #9 `@cds.persistence.journal` | ACCEPTED WITH MODIFICATION | CAP-DB-007 | Added the documented "must be checked into the version control system" requirement for `.hdbmigrationtable` files |
| #10 Static model style (Java) | DOWNGRADED | CAP-DB-010 | Medium → **Low** ("recommended" style guidance; both styles supported) |
| CAP-PERF #6 Decimal/Int64 arithmetic in DB (cross-category) | MERGED | → CAP-DB-009 | Relocated: silent precision loss is a data-integrity concern, not (only) performance; documented string-return behavior verified verbatim |
| — (batch scope: concurrency/locking) | NEW RULE | CAP-DB-008 | Framework locking primitives (`forUpdate`/`.lock()` incl. `SKIP_LOCKED`) vs hand-rolled lock flags; documented limitations verified (no projections/views; no SQLite; H2 exclusive-only) |

### Transaction candidates (5 reviewed + 1 cross-category pull)

| Candidate (§CAP-TXN) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Managed transactions in handlers | ACCEPTED | CAP-TXN-001 | "You don't have to care" (Node) and auto-ChangeSet (Java) verified verbatim |
| #2 `cds.tx()` outside handlers; functional form | ACCEPTED | CAP-TXN-002 | SQLite parallel-transaction deadlock statement verified and embedded |
| #3 No `cds.tx(req)` | DOWNGRADED | CAP-TXN-003 | ⚠ Docs do **not** say "deprecated" — actual wording: "still works but is not required nor recommended anymore" → authority REQ → **SAP-REC**, severity Medium → **Low** (stale idiom, still functional) |
| #4 ChangeSet API / Spring `@Transactional` | ACCEPTED WITH MODIFICATION | CAP-TXN-004 | Added raw-JDBC-transaction-control prohibition on the CAP datasource and the `enforceTransactional=false` default note |
| #5 No distributed-atomicity assumptions | ACCEPTED WITH MODIFICATION | CAP-TXN-005 | ⚠ Re-sourced: the "not as a distributed atomic transaction" quote exists only on the **Node.js** cds-tx page; the Java changeset page never mentions atomicity — rule reworded to "documented limitation (Node) + no documented guarantee (Java)", High kept |
| CAP-LOGIC #5 Always await emits/async ops (cross-category) | MERGED | → CAP-TXN-006 | Relocated: a transaction-integrity requirement ("invalid transaction states and deadlocks" verified verbatim); High, SAP-REQ, Node.js |

### Event / messaging candidates (7 reviewed)

| Candidate (§CAP-EVT) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Protocol-agnostic events | DOWNGRADED | CAP-EVT-001 | High → **Medium**: direct violation is coupling/duplication; escalations route through CAP-EVT-002/CAP-MT-003/CAP-ARCH-002 |
| #2 Persistent outbox in production | ACCEPTED WITH MODIFICATION | CAP-EVT-002 | ⚠ Authority REQ → **SAP-REC**: the queue is the documented default with documented rationale, but `cds.unqueued`/immediate dispatch is a documented API — the rule governs *undocumented disabling*, with the escape hatch requiring recorded justification. **Critical kept** (justified: phantom/lost events = cross-system inconsistency + lost audit evidence, in SAP's own terms). cds 8→10 double-processing hazard embedded (verified on the live page) |
| #3 Idempotent handlers | ACCEPTED | CAP-EVT-003 | "Handlers must still be idempotent" verified verbatim (SAP-REQ); **Critical kept** with explicit justification (duplicate business effects = silent data corruption; contrast with High CAP-MT-004 explained in the rule) |
| #4 No sensitive data in headers; claims in payload | DOWNGRADED | CAP-EVT-004 | Critical → **High**: requires specific mistakes rather than default-open. ⚠ Nuance corrected: SAP allows claims "via payload or headers at queue time" — the prohibition is *sensitive* data (tokens/PII/secrets) in outbound headers; privileged-mode role-check failure mode added |
| #5 Dead-letter operations | ACCEPTED WITH MODIFICATION | CAP-EVT-005 | ⚠ Corrected: 4xx-skip-retry is a **developer-implemented** pattern (`e.unrecoverable = true` / `setCompleted()`), not automatic; framework auto-detects only some unrecoverable errors. Revive/delete service pattern + OTel metrics verified; thresholds stay ORG (G-32) |
| #6 Event Hub default broker | ACCEPTED | CAP-EVT-006 | "The new default offering" verified verbatim; broker matrix stays gap G-31 |
| #7 Local messaging for local dev | DOWNGRADED | CAP-EVT-007 | Medium → **Low** (developer experience); ⚠ tunneling-service qualifier added per exact wording |

### Batch summary

| Metric | Data | Transactions | Events |
|---|---|---|---|
| Candidates reviewed | 10 (9 newly dispositioned) | 5 (+1 cross-category) | 7 |
| Accepted unchanged | 3 | 2 | 2 |
| Accepted with modification | 3 | 2 | 3 |
| Merged | 1 out-of-inventory absorbed (#4) + CAP-PERF #6 in | CAP-LOGIC #5 in | 0 |
| Downgraded | 2 | 1 | 3 |
| Rejected | 0 | 0 | 0 |
| Deferred | 0 | 0 | 0 |
| New rules | 1 (CAP-DB-008) | 0 | 0 |
| **Final rules** | **10** | **6** | **7** |

Batch severity: **2 Critical / 9 High / 9 Medium / 3 Low**. Both Critical rules (CAP-EVT-002, CAP-EVT-003) carry explicit justifications tied to SAP's own documented failure vocabulary (phantom/lost events; must-be-idempotent). Authority: 7 SAP-REQ, 16 SAP-REC. Runtime: 7 Node.js-only (CAP-DB-003/-005/-006/-009, CAP-TXN-002/-003/-006), 2 Java-only (CAP-DB-010, CAP-TXN-004), 14 Both.

### Verification corrections applied in this batch (summary)

1. `cds.tx(req)`: "not required nor recommended", not "deprecated" (CAP-TXN-003).
2. Distributed atomicity: Node-only quote; Java silent — reworded (CAP-TXN-005).
3. 4xx retry skipping: developer-implemented, not framework-automatic (CAP-EVT-005).
4. Event Hub localhost limitation: "unless you use a tunneling service" qualifier (CAP-EVT-007).
5. `/docs/node.js/queue` is 404 — the event-queues guide is the citable source (source map updated); Node queue API is `cds.queued()`/`cds.unqueued()`.
6. `@cap-js` migration authority: "highly recommended" = SAP-REC, no removal date (CAP-DB-003).

### Scope items reviewed with no rule created (Batch 3)

- **Mandating the outbox pattern universally** — deliberately NOT done: the queue is the default and CAP-EVT-002 protects the default; SAP documents immediate dispatch as a legitimate API, so mandating outbox-everywhere would exceed the evidence.
- **Exactly-once delivery** — no rule can assert it: SAP explicitly documents the opposite; treated via idempotency (CAP-EVT-003).
- **Event ordering guarantees** — SAP documents none on the verified pages; no rule authored (would manufacture a guarantee); ordering-sensitive designs fall under CAP-TXN-005's partial-failure design duty.
- **Bulk operations / batch sizing** — Java documents a max-batch-size default; tuning is performance material → future CAP-PERF.
- **Indexes** — no CAP-level index guidance found on the verified pages beyond performance modeling → future CAP-PERF.
- **Retry tuning values** (maxAttempts/backoff) — documented defaults exist; concrete tuning is ORG operations policy (G-32/G-33), not a rule.

---

## Batch 4 — Business Logic & Integration (2026-08-11)

Verification basis: one targeted verification pass (handler mechanics; remote-service consumption and resilience) against live official documentation on 2026-08-11. Corrections marked ⚠. Final rules: [business-logic.md](../standards/rules/business-logic.md) (5), [integration.md](../standards/rules/integration.md) (7).

### Business-logic candidates (6 remaining after Batches 2–3 consumed #1 and #5)

| Candidate (candidate-rules.md §CAP-LOGIC) | Disposition | Final rule | Notes |
|---|---|---|---|
| #2 Correct phase usage | ACCEPTED WITH MODIFICATION | CAP-LOGIC-002 | Severity High → **Medium**; absorbs #4. ⚠ Verification nuances embedded: `before`/`after` "always executed concurrently" (Node); `on` sequential **for requests only** — concurrent for async events; phase-*purpose* framing is derivation from documented mechanics, labeled accordingly |
| #3 `return super.init()` | DOWNGRADED | CAP-LOGIC-003 | High → **Medium** (loud-but-confusing failure, test-catchable). ⚠ Correction: SAP requires *calling* `super.init()`; its position is a documented handler-ordering choice, not a mandate to place it last |
| #4 Collect errors via `req.error` | MERGED | → CAP-LOGIC-002 | Validation-*timing* concern folded into the phase rule; error-response mechanics stay with the future CAP-ERR category (canonical source re-verified: node.js/events) |
| #6 Java handler registration + typed contexts | DOWNGRADED | CAP-LOGIC-004 | High → **Medium**; `EventHandler` interface "is required" (REQ element), typed contexts "whenever possible", async `@On` completion "not recommended" — all verified verbatim |
| #7 Actions need `@On` handlers in Java | MERGED | → CAP-SRV-004 | Already covered by Batch 2 (runtime field + detection step 5); not re-authored |
| #8 Java service layering | ACCEPTED | CAP-LOGIC-005 | "Should be registered on an Application Service" verified verbatim (SAP-REC); Persistence Service scoped to technical concerns per SAP's own example |
| — (batch scope: logic-placement anchor) | NEW RULE | CAP-LOGIC-001 | Domain behavior in event handlers on the owning service; grounded in custom-code (handlers as the documented mechanism) + concepts (determinations in event handlers); no verbatim "must" sentence exists, hence SAP-REC |

### Integration candidates (7 reviewed)

| Candidate (§CAP-INT) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 `cds import`, never copy CDS files | ACCEPTED WITH MODIFICATION | CAP-INT-001 | Absorbs #2. ⚠ Verification *upgrade*: "Always use OData V4 (`odata`) when calling another CAP service" is REQ-grade wording; the three documented copy-problems verified verbatim (incl. "CAP creates unneeded database tables and views") |
| #2 OData V4 / EDMX between services | MERGED | → CAP-INT-001 | Same concern (exchange format + protocol) |
| #3 Consumption views | DOWNGRADED | CAP-INT-003 | High → **Medium**: "always prefer" verified, but views are formally optional; coupling/maintainability concern |
| #4 No credentials in destination config | MERGED | → CAP-SEC-017 | Already covered by Batch 1 (SEC-017 cites the consuming-services warning); not re-authored |
| #5 Mock remote services | ACCEPTED | CAP-INT-004 | Mechanics verified (auto-mock with `cds watch`, `cds mock`, CSV in `srv/external/data`); Java `--with-mocks` is REQ-phrased ("make sure to add") |
| #6 Cross local/remote expands | ACCEPTED | CAP-INT-005 | High kept: documented stability warning + URL-length mechanism verified; "Consider to reject expands… on a list of items" is the documented advice (SAP-REC) |
| #7 MT destination retrieval strategy | ACCEPTED | CAP-INT-006 | Subscriber-tenant default + `alwaysProvider`/`AlwaysProvider` override verified on both runtime pages |
| — (batch scope: consumption mechanism) | NEW RULE | CAP-INT-002 | Remote systems via CAP remote services + destinations; REQ elements verified: Java Remote Services "need to be configured explicitly" / never auto-created; on-premise requires Connectivity service + Cloud Connector. ⚠ Correction: "Destination service recommended for productive use" is NOT a quotable SAP sentence — phrased as the documented mechanism instead |
| — (batch scope: reliability) | NEW RULE | CAP-INT-007 | Design remote calls for failure. Guarantee discipline: CAP Node.js has "no resilience library provided out of the box"; the Java Remote Services page documents **no** timeout/retry options (verified absence); SAP names options (service mesh, Java Cloud SDK `ResilienceDecorator`, Node community packages) without prescribing — duty = SAP-REC, mechanism choice = GEN, thresholds = ORG (G-27) |

### Batch summary

| Metric | Business Logic | Integration |
|---|---|---|
| Candidates reviewed | 6 | 7 |
| Accepted unchanged | 1 | 3 |
| Accepted with modification | 1 | 1 |
| Merged | 2 (#4 internal; #7 → CAP-SRV-004) | 2 (#2 internal; #4 → CAP-SEC-017) |
| Downgraded | 2 | 1 |
| Rejected | 0 | 0 |
| Deferred | 0 | 0 |
| New rules | 1 (CAP-LOGIC-001) | 2 (CAP-INT-002, CAP-INT-007) |
| **Final rules** | **5** | **7** |

Batch severity: **0 Critical / 3 High / 9 Medium / 0 Low** (High: CAP-INT-001 shadow-persistence integrity, CAP-INT-002 auth/tenant-adjacent bypass, CAP-INT-005 production stability). Authority: 2 SAP-REQ (CAP-LOGIC-003, CAP-INT-001), 10 SAP-REC. Runtime: 1 Node.js-only (CAP-LOGIC-003), 2 Java-only (CAP-LOGIC-004/-005), 9 Both.

### Scope items reviewed with no rule created (Batch 4)

- **Generic layered-architecture prescriptions for business logic** — deliberately not imposed: CAP documents handlers-on-services, not layer counts (consistent with Batch 2's rejection).
- **Handler error-response mechanics** (`req.error`/`reject` rendering, i18n, sanitization) — deferred to the future CAP-ERR category; only validation *timing* is governed here (CAP-LOGIC-002).
- **Business-logic idempotency/side-effect rules** — owned by CAP-EVT-003 (async) and CAP-TXN-005 (partial failure); cross-referenced, not restated.
- **Batch requests to remote systems / payload-mapping conventions** — no normative SAP guidance found beyond mechanics; mapping style is design, not rule material.
- **Circuit-breaker mandates and concrete timeout values** — SAP names no thresholds and prescribes no mechanism; remains ORG gap G-27 (CAP-INT-007 enforces only the deliberate-decision duty).
- **Integration-testing depth** — future CAP-TEST category; CAP-INT-004 covers only mockability of the inner loop.

---

## Batch 5 — Testing, Error Handling & Observability (2026-08-12)

Verification basis: one targeted verification pass (cds-test / Java testing; Node & Java error pages; logging/observability pages) against live official documentation on 2026-08-12. Corrections marked ⚠. Final rules: [testing.md](../standards/rules/testing.md) (7), [error-handling.md](../standards/rules/error-handling.md) (6), [logging-observability.md](../standards/rules/logging-observability.md) (5).

### Testing candidates (7 reviewed, 1 new rule)

| Candidate (candidate-rules.md §CAP-TEST) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Bootstrap with `cds.test()` first | ACCEPTED WITH MODIFICATION | CAP-TEST-001 | Absorbs #2 (isolation); severity High → **Medium** (test infrastructure, not production). Bootstrap-order wording verified REQ-strength ("always ensure … goes first") |
| #2 In-memory SQLite + `data.reset()` | MERGED | → CAP-TEST-001 | "Can be used" wording → REC element of the same rule |
| #3 Assert stable codes, not texts/snapshots | ACCEPTED | CAP-TEST-003 | `containSubset` confirmed as SAP's named alternative; Medium kept (brittleness erodes evidence value) |
| #4 Runner-portable tests | ACCEPTED | CAP-TEST-004 | ⚠ Correction: **no "Jest deprecated/abandoned" statement exists** — docs note only ESM/Chai friction; the inventory's cds-10 claim dropped from the rule |
| #5 Java layered testing | DOWNGRADED | CAP-TEST-002 | High → **Medium**; ⚠ H2 wording corrected to "support **until** version 2.3.x"; MTX-not-on-H2 verified as hard limitation |
| #6 Mock users for authenticated flows | ACCEPTED | CAP-TEST-005 | ⚠ Correction: no CAP `@MockUser` annotation documented — Java side is Spring Security's `@WithMockUser` + CAP mock-user config |
| #7 Hybrid tests via `cds bind --exec` | ACCEPTED | CAP-TEST-006 | Reframed as coverage duty for parity-sensitive features (links CAP-DB-002/-008, CAP-TEST-002) |
| — (batch scope: security verification) | NEW RULE | CAP-TEST-007 | **High.** Origin: SAP's documented instruction to add authentication tests for deployed unauthenticated rejection (verified Batch 1) + the verification duty for the CAP-SEC/CAP-MT rules; role-matrix element labeled GEN. Layer 2 counterpart of AI-TEST-004 |

### Error-handling candidates (7 reviewed)

| Candidate (§CAP-ERR) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 `req.error`/`req.reject`, not raw throws | ACCEPTED WITH MODIFICATION | CAP-ERR-001 | Absorbs #5 into one both-runtime rule; severity High → **Medium** (API quality/diagnosability) |
| #2 Never disable 5xx sanitization | DOWNGRADED | CAP-ERR-002 | Critical → **High** (disclosure aiding attack, not direct access — consistent with CAP-SEC-016 calibration). ⚠ Runtime scoped **Node.js**: Java documents only behavioral generic-500 fallback, no security-worded sanitization — Java duty reworded to "no internals in ServiceException messages" |
| #3 Fail loudly ("let it crash") | ACCEPTED | CAP-ERR-003 | High kept (SAP's own multitenant information-disclosure rationale verified verbatim); authority SAP-REC (Node source) with GEN note for Java |
| #4 `srv.on('error')` synchronous only | DOWNGRADED | CAP-ERR-004 | High → **Medium** (scoped misuse; "expected to be a sync function, that is, not `async`" verified) |
| #5 `ServiceException`+`ErrorStatuses`/`Messages` | MERGED | → CAP-ERR-001 | Same concern, Java mechanics |
| #6 Localized codes with targets | ACCEPTED | CAP-ERR-005 | Medium kept; Fiori `target` interpretation verified |
| #7 Validate user input in messages | DOWNGRADED | CAP-ERR-006 | Critical → **High** (reflected injection channel, consumer-dependent exploitation); both-runtime warnings verified verbatim (Java adds URLs) |

### Logging/observability candidates (6 reviewed)

| Candidate (§CAP-LOG) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 `cds.log`/SLF4J, parameterized | ACCEPTED WITH MODIFICATION | CAP-LOG-001 | Absorbs #4 (per-component levels); severity High → **Medium**; sensitive-content aspects explicitly left with CAP-SEC-016 |
| #2 JSON structured logs in production | DOWNGRADED | CAP-LOG-002 | High → **Medium** (diagnostics capability); Node default-since-7.5 still stated on current docs; Java = active setup via `cf-java-logging-support` profile split (verified) |
| #3 Correlation-ID propagation | ACCEPTED | CAP-LOG-003 | Automatic behavior verified on both runtimes; rule = don't break it in custom code |
| #4 Per-component levels; masking on | MERGED | → CAP-LOG-001 (levels) / CAP-SEC-016 (masking & production levels — already owned there since Batch 1) | Split by concern to avoid duplication |
| #5 OpenTelemetry via SAP tooling | ACCEPTED | CAP-LOG-004 | ⚠ No "Telemetry v2" status statement on the docs — version claim dropped; rule scoped to *mechanism when adopted* (adoption = ORG/CAP-OPS, G-37) |
| #6 Expose only health endpoints publicly | DOWNGRADED | CAP-LOG-005 | Critical → **High**; ⚠ authority corrected to **SAP-REC** per verified wording ("For security reasons, it's recommended…"); runtime scoped **Java** (actuator mechanism; Node has only the built-in `/health`) |

### Batch summary

| Metric | Testing | Error handling | Logging |
|---|---|---|---|
| Candidates reviewed | 7 | 7 | 6 |
| Accepted unchanged | 4 | 2 | 2 |
| Accepted with modification | 1 | 1 | 1 |
| Merged | 1 (#2) | 1 (#5) | 1 (#4, split two ways) |
| Downgraded | 1 | 3 | 2 |
| Rejected | 0 | 0 | 0 |
| Deferred | 0 | 0 | 0 |
| New rules | 1 (CAP-TEST-007) | 0 | 0 |
| **Final rules** | **7** | **6** | **5** |

Batch severity: **0 Critical / 5 High / 12 Medium / 1 Low** (High: CAP-TEST-007, CAP-ERR-002/-003/-006, CAP-LOG-005). Authority: 4 SAP-REQ (CAP-TEST-001 bootstrap, CAP-ERR-002/-004/-006), 14 SAP-REC. Runtime: 5 Node.js-only (CAP-TEST-001/-003/-004, CAP-ERR-002/-004), 2 Java-only (CAP-TEST-002, CAP-LOG-005), 11 Both. No coverage percentages, pyramid ratios, monitoring SLAs, or error taxonomies were invented — those remain gaps G-01/G-02/G-37.

### Verification corrections applied (summary)

1. No "Jest deprecated" claim exists — dropped (CAP-TEST-004).
2. H2 support wording: "until version 2.3.x" (CAP-TEST-002).
3. Java mock users: Spring `@WithMockUser`, not a CAP annotation (CAP-TEST-005).
4. 5xx sanitization is Node-only as a security requirement; Java has behavioral generic-500 wording only (CAP-ERR-002 runtime scoping).
5. Actuator exposure is SAP-REC ("recommended … for security reasons"), severity justified independently (CAP-LOG-005).
6. `@cap-js/telemetry` version status unverifiable on docs — no version claim (CAP-LOG-004).

### Scope items reviewed with no rule created (Batch 5)

- **Coverage thresholds / test-pyramid ratios** — SAP prescribes none; remains gap G-01 (deliberately not manufactured).
- **Error taxonomy (client/business/technical)** — SAP documents mechanics, no taxonomy; remains gap G-02; CAP-ERR-001/-005 require stable codes without prescribing a scheme.
- **Log retention, audit-vs-application log policy** — ORG gaps G-03/G-06 unchanged; the audit-logging *mechanism* stays with the future CAP-PRIV batch.
- **Monitoring/alerting mandates, SLAs, probe intervals** — gap G-37; CAP-LOG-004 governs mechanism-when-adopted only.
- **Health-check wiring** — CAP-OPS/CAP-DEP candidate territory (deployment descriptors), untouched here.

---

## Batch 6 — Performance, Extensibility & Privacy (2026-08-12)

Verification basis: one targeted verification pass (performance-modeling and served-ootb pages; extensibility pages; all five privacy pages with explicit reachability checks) on 2026-08-12. Corrections marked ⚠. Final rules: [performance.md](../standards/rules/performance.md) (7), [extensibility.md](../standards/rules/extensibility.md) (4), [data-privacy.md](../standards/rules/data-privacy.md) (4).

### Performance candidates (8 in inventory; #6 relocated in Batch 3)

| Candidate (candidate-rules.md §CAP-PERF) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Intentional pagination limits | MERGED | → CAP-SEC-014 | The deliberate-limit decision (incl. keeping the default cap, never `limit: 0`) is already normative there since Batch 1 — not re-authored |
| #2 Reliable pagination (OData V4) | ACCEPTED | CAP-PERF-001 | All documented limitations verified verbatim, incl. the sensitive-data skip-token warning |
| #3 Associations + `$expand` over JOIN views | DOWNGRADED | CAP-PERF-002 | High → **Medium** (chronic cost, not explosion); "A JOIN will not be executed until you explicitly use `$expand`" + detail-first pattern verified |
| #4 No live-calculated fields in filter/sort | ACCEPTED | CAP-PERF-003 | **High kept** with justification (full-table scans on the normal request path at scale — the named resource-exhaustion case); preference order and `@Capabilities` guidance verified verbatim |
| #5 Avoid UNIONs; remodel polymorphism | ACCEPTED | CAP-PERF-004 | Both documented remodeling options verified (discriminator+compositions; sparse aspect table) |
| #6 Decimal/Int64 arithmetic in DB | (Batch 3) MERGED | → CAP-DB-009 | Historical |
| #7 `srv.foreach` for large result sets | ACCEPTED | CAP-PERF-005 | Exact wording verified ("instead of materializing the full result set in memory") |
| #8 Single-field primary keys | MERGED | → CAP-CDS-002 | The JOIN-performance rationale is absorbed into the key rule's scope (related-rules line updated); a standalone Low rule would duplicate the cuid preference |
| #9 Composition-tree size awareness | ACCEPTED | CAP-PERF-006 | "Copied entirely into draft state, even when only one little part is changed" verified verbatim |
| — (gap G-05: N+1 in handlers) | NEW RULE | CAP-PERF-007 | **High, GEN authority** — SAP documents the set-based primitives, not the prohibition; authored as the resolution of gap G-05 with honest authority labeling. Remote fan-out variant remains CAP-INT-005 |

### Extensibility candidates (3 reviewed, 1 new rule)

| Candidate (§CAP-EXT) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Extension allowlist as deliberate artifact | ACCEPTED | CAP-EXT-002 | High kept (base-model integrity for all tenants); `x_` prefix requirement and `cds build`-validation-aborts-`cds push` verified verbatim; limit values stay ORG (G-35) |
| #2 Extension workflow | ACCEPTED | CAP-EXT-003 | `cds login`/`pull`/`push`, test-tenant-first, and `cds.ExtensionDeveloper` role verified |
| #3 Feature-toggle constraints | DOWNGRADED | CAP-EXT-004 | High → **Medium** (violations fail loudly at build/first toggle-combination test); all three limitations verified in the docs' warning box; ⚠ production-provider asymmetry embedded (Java Feature Toggles Info Provider required; Node.js has "no out-of-the-box feature toggles provider for production yet") |
| — (batch scope: extend-vs-modify anchor) | NEW RULE | CAP-EXT-001 | ⚠ Citation corrected: intrinsic-extensibility statements live on **get-started/concepts** (`/docs/about/best-practices` is 404); upgrade-safety anchor: no in-place modification of reuse/base/generated artifacts |

### Privacy candidates (4 reviewed + deferred CAP-SEC #17)

| Candidate | Disposition | Final rule | Notes |
|---|---|---|---|
| **CAP-SEC #17 (deferred from Batch 1)** | **MERGED** | → CAP-PRIV-001 | Resolved as planned: the `@PersonalData` annotation duty is owned by Privacy; Security cross-references (SEC-016's audit-log pointer now resolves here). Not recreated as a Security rule |
| CAP-PRIV #1 Complete `@PersonalData` semantics | ACCEPTED WITH MODIFICATION | CAP-PRIV-001 | Absorbs #2 and SEC #17. ⚠ Two wording corrections: DataSubjectID strength is "needs to identify" (REQ substance, no "must" quote); the "apply IsPotentiallySensitive selectively" advice is **not SAP wording** — reframed as our operational rationale (every read audited), explicitly labeled |
| CAP-PRIV #2 Annotations in dedicated file | MERGED | → CAP-PRIV-001 | "Following the best practice of separation of concerns… srv/data-privacy.cds" verified — folded as the documented pattern |
| CAP-PRIV #3 Audit logging plugin + service | ACCEPTED | CAP-PRIV-002 | Automatic logging of personal-data changes/sensitive reads, old/new values, outbox-by-default, and production `audit-log-to-restv2` verified; ⚠ Java specifics are a documented **pointer** to `/docs/java/auditlog` — rule instructs verifying there rather than asserting details |
| CAP-PRIV #4 PDM flat + role-protected | ACCEPTED | CAP-PRIV-003 | High kept (concentrated personal-data surface); flat-structures need, `PersonalDataManagerUser` grant to the PDM instance, and enterprise-account constraint verified verbatim |
| — (batch scope: retention/erasure) | NEW RULE | CAP-PRIV-004 | **GEN authority, no SAP mandate claimed:** ⚠ the DRM guide is explicitly "Under construction" with all substantive content unrendered (HTML comments) — verified 2026-08-12; no reachable statement requires DRM integration. The rule mandates a *documented approach* (inventory-mapped erasure incl. drafts/outbox/managed fields), periods stay ORG/legal (G-16), mechanism state in-flux (G-17) |

### Batch summary

| Metric | Performance | Extensibility | Privacy |
|---|---|---|---|
| Candidates reviewed | 8 (7 newly dispositioned) | 3 | 4 (+ deferred SEC #17) |
| Accepted unchanged | 4 | 2 | 2 |
| Accepted with modification | 0 | 0 | 1 |
| Merged | 2 (#1 → SEC-014; #8 → CDS-002) | 0 | 2 (#2; SEC #17) |
| Downgraded | 1 | 1 | 0 |
| Rejected | 0 | 0 | 0 |
| Deferred | 0 | 0 | 0 |
| New rules | 1 (CAP-PERF-007, GEN, from G-05) | 1 (CAP-EXT-001) | 1 (CAP-PRIV-004, GEN) |
| **Final rules** | **7** | **4** | **4** |

Batch severity: **0 Critical / 6 High / 9 Medium / 0 Low** (High: PERF-003/-007, EXT-002, PRIV-001/-002/-003). Authority: 3 SAP-REQ (EXT-002, EXT-004, PRIV-003), 10 SAP-REC, 2 GEN (PERF-007, PRIV-004 — the catalog's first GEN rules, both with explicitly documented non-SAP origins). Runtime: 1 Node.js-only (PERF-005), 14 Both.

### Verification corrections applied (summary)

1. Intrinsic-extensibility citation moved to get-started/concepts (`/docs/about/best-practices` 404) — CAP-EXT-001.
2. DataSubjectID strength: "needs to identify", not "must" — CAP-PRIV-001.
3. "Selective sensitive-tagging" is not SAP wording — reframed as labeled rationale — CAP-PRIV-001.
4. DRM guide under construction, content unrendered — CAP-PRIV-004 authored as GEN with zero SAP authority claimed.
5. Java audit-logging details are a pointer to `/docs/java/auditlog` — CAP-PRIV-002 instructs per-runtime verification.
6. Node.js production feature-toggle provider does not exist out of the box — embedded in CAP-EXT-004.

### Scope items reviewed with no rule created (Batch 6)

- **Performance budgets, SLOs, sizing, caching mandates** — SAP prescribes none; gap G-04 stays open (M0 NFR material). CAP documents no general application-cache mechanism to mandate — no caching rule invented.
- **Database index guidance** — no CAP-level index rules found beyond the calculated-fields/index interaction (embedded in PERF-003); native index design is database documentation, not CAP.
- **In-app vs side-by-side extension strategy selection** — the four documented categories are described, not prescribed per situation; strategy choice is architecture (ADR per CAP-ARCH-007).
- **Data masking** — no CAP-documented general masking mechanism beyond log-header masking (CAP-SEC-016) and the audit mechanisms; nothing to formalize.
- **Retention periods / legal grounds** — G-16 unchanged (legal, not engineering).

---

## Batch 7 (final) — Deployment, CI/CD, Versions & Operations (2026-08-12)

Verification basis: two passes on 2026-08-12 — a **full version-baseline re-verification** (headline: the June 2026 wave is still current; the August 2026 minor cds 10.1 / CAP Java 5.1 is listed but explicitly UNRELEASED) and a deployment/CI-CD/operations pass; plus a direct re-check of the may25 release note for the `hdbcds` claim. [docs/version-management.md](../docs/version-management.md) rewritten to the 2026-08-12 baseline. Final rules: [deployment.md](../standards/rules/deployment.md) (3), [cicd.md](../standards/rules/cicd.md) (3), [versions-dependencies.md](../standards/rules/versions-dependencies.md) (6), [production-readiness.md](../standards/rules/production-readiness.md) (3). **Phase 2 rule authoring is complete — all 20 categories authored.**

### Deployment candidates (6 reviewed)

| Candidate (candidate-rules.md §CAP-DEP) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 MTA-based CF deployment | ACCEPTED WITH MODIFICATION | CAP-DEP-001 | Severity High → **Medium** (mechanism/maintainability; dangerous *content* is owned elsewhere); ⚠ UI facet list corrected per verification (add `app-frontend`; `portal` = multitenant) |
| #2 Production facets before first deploy | MERGED | → CAP-SEC-002 / CAP-DB-001/-002 | The "no SQLite/mocked auth in production" content is already Critical/High normative there; DEP-001 keeps the facets as the scaffold path |
| #3 Landscape config in `.mtaext` | ACCEPTED | CAP-DEP-002 | "Allows you to keep landscape-specific deployment settings outside your base mta.yaml" verified |
| #4 Kyma Helm + buildpacks | DOWNGRADED | CAP-DEP-003 | High → **Medium**; buildpack properties (reproducible, unprivileged, SBoM) and values.yaml preservation verified verbatim; absorbs #6 |
| #5 `cds bind` hybrid, no materialized credentials | MERGED | → CAP-SEC-017 | Already normative there since Batch 1 (pointer-only bindings) |
| #6 Read-only pull secrets | MERGED | → CAP-DEP-003 | Imperative wording verified ("use a technical user with read-only permissions") — folded as a strong clause |

### CI/CD candidates (3 reviewed, 1 new rule)

| Candidate (§CAP-CICD) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Pipelines scaffolded via `cds add github-actions` | ACCEPTED WITH MODIFICATION | CAP-CICD-001 | Reframed as capability rule ("build/test/deploy through a pipeline") with the documented scaffolds preferred — no product prescribed (GitHub Actions worked example; SAP CI/CD service and Piper as documented alternatives); High kept (laptop deploys = unreproducible production) |
| #2 Protected environment + tagged releases | ACCEPTED WITH MODIFICATION | CAP-CICD-002 | Added artifact attribution and the rollback-scope honesty clause (**no DB schema rollback guarantees invented** — schemas roll forward per CAP-DB-007) |
| #3 Cloud-backed integration tests via `cds bind --exec` in CI | MERGED | → CAP-TEST-006 | Hybrid CI execution already normative there (Batch 5) |
| — (batch scope: quality gates; Phase 3 enabler) | NEW RULE | CAP-CICD-003 | **ORG** — SAP prescribes no gate set (G-39 keeps the open specifics); defines the four enforcement classes (automatic / manual review / deployment-time / operational) that Phase 3 will map rules onto |

### Version candidates (9 reviewed)

| Candidate (§CAP-VER) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Freeze dependencies via lockfile | ACCEPTED WITH MODIFICATION | CAP-VER-001 | Absorbs #2 (refresh half of the same lifecycle); severity Critical → **High** (probabilistic drift, not direct exploit); "should freeze all their dependencies, including transient ones" verified verbatim |
| #2 Regular refresh (Dependabot/Renovate one-by-one) | MERGED | → CAP-VER-001 | "One-by-one" wording verified; SLA stays ORG (G-40) |
| #3 Active CAP major, maintenance window | DOWNGRADED | CAP-VER-002 | Critical → **High** (consistent with CAP-MT-001's unsupported-≠-breached calibration); absorbs #4; maintenance-model wording re-verified. ⚠ cds 9's Maintenance status is policy-derived, not verbatim — flagged in the baseline |
| #4 Latest minors monthly / patches ASAP (Java) | MERGED | → CAP-VER-002 | The consumption-cadence clause |
| #5 Current runtime baselines | DOWNGRADED | CAP-VER-003 | Critical → **High**; rule binds the **live baseline document** instead of hardcoded numbers; adds the pin-consistency requirement (documented update-every-pin-location duty) |
| #6 Never mix CAP package major lines | DOWNGRADED | CAP-VER-004 | Critical → **High** (most mixes fail loudly; the subtle instance-level case is documented and owned at CAP-EVT-002) |
| #7 Official migration tooling | DOWNGRADED | CAP-VER-005 | High → **Medium** (process quality; breakage classes owned by referenced rules); `cds upgrade` alpha status preserved |
| #8 `hdbtable` not `hdbcds` | DOWNGRADED | CAP-VER-006 | High → **Medium** (loud build/deploy failure; review value = catching legacy config pre-upgrade). ⚠ Claim re-verified **directly on the live may25 release note** ("can now no longer be used") after the baseline agent could not find it on the HANA guide; HANA Cloud never used hdbcds — applicability narrowed accordingly |
| **#9 Queue drain on cds 8→10** | **MERGED** | → **CAP-EVT-002** (version note) | Explicit disposition as required: the hazard is documented on the event-queues page and has been owned by CAP-EVT-002's CAP-version field since Batch 3 (re-verified verbatim 2026-08-12). CAP-VER-005 references release-note operational steps generically — **single authoritative requirement, no duplicate** |

### Operations candidates (2 remaining + 2 historical + go-live scope)

| Candidate (§CAP-OPS) | Disposition | Final rule | Notes |
|---|---|---|---|
| #1 Health probes wired in descriptors | DOWNGRADED | CAP-OPS-001 | High → **Medium** (degraded lifecycle management, loud under failure); ⚠ facet list corrected: probes wired by `mta` and `kyma` facets (cf-manifest not confirmed — not cited); CF readiness-tooling caveat re-verified ("not yet supported") |
| #2 Production UI entry point | ACCEPTED | CAP-OPS-002 | "Aren't available in the cloud" verified verbatim; facet list updated (incl. `app-frontend`) |
| #3 MCP exposure governance | (Batch 1) MERGED | → CAP-SEC-018 | Historical — **not recreated** (MCP adapter status re-verified 2026-08-12: still Beta; Java flavor still not public) |
| #4 Topology by configuration | MERGED | → CAP-ARCH-006 | The config-not-code topology mechanism is that rule's documented rationale; a standalone rule would duplicate it |
| — (batch scope: G-36 go-live checklist) | NEW RULE | CAP-OPS-003 | **ORG, High** — closes gap G-36 by binding the M9 lifecycle gate as a rule: aggregation-and-evidence only (every element cites its owning rule; nothing double-normed); the operational drill (trace-through, probes, runbook) makes M9 concretely assessable |

### Batch summary

| Metric | Deployment | CI/CD | Versions | Operations |
|---|---|---|---|---|
| Candidates reviewed | 6 | 3 | 9 | 2 (+2 historical) |
| Accepted unchanged | 1 | 0 | 0 | 1 |
| Accepted with modification | 1 | 2 | 1 | 0 |
| Merged | 3 (#2, #5 out; #6 in) | 1 (#3 → CAP-TEST-006) | 3 (#2, #4 in; **#9 → CAP-EVT-002**) | 1 (#4 → CAP-ARCH-006) |
| Downgraded | 1 | 0 | 5 | 1 |
| Rejected | 0 | 0 | 0 | 0 |
| Deferred | 0 | 0 | 0 | 0 |
| New rules | 0 | 1 (CAP-CICD-003, ORG) | 0 | 1 (CAP-OPS-003, ORG) |
| **Final rules** | **3** | **3** | **6** | **3** |

Batch severity: **0 Critical / 6 High / 9 Medium / 0 Low** (High: CAP-CICD-001, CAP-VER-001..004, CAP-OPS-003). Authority: 5 SAP-REQ (CAP-VER-001/-002/-003/-004/-006), 8 SAP-REC, 2 ORG (CAP-CICD-003, CAP-OPS-003). Runtime: 2 Node.js-only (CAP-VER-001/-004), 13 Both.

### Verification corrections applied (summary)

1. **Version baseline re-verified in full (2026-08-12): June 2026 wave still current; Aug 2026 minor (10.1/5.1) listed but UNRELEASED** — version-management.md rewritten with today's date, not-established flags recorded honestly (`@cap-js/*` patch versions unpublished; cds 9 Maintenance policy-derived).
2. `hdbcds` removal re-sourced to the live may25 release note after the HANA guide no longer carries it.
3. UI facets corrected (add `app-frontend`); health-probe facets = `mta` + `kyma` only.
4. No zero-downtime/blue-green rules — SAP's deployment pages document none (verified); ORG territory (G-39).
5. MCP adapter re-verified still Beta (CAP-SEC-018's status current).

### Scope items reviewed with no rule created (Batch 7)

- **Zero-downtime/blue-green deployment** — no SAP guidance exists on the deployment pages (verified); ORG strategy (G-39).
- **Database migration rollback guarantees** — deliberately NOT invented; schemas roll forward (CAP-DB-007); CAP-CICD-002 scopes rollback to the application artifact.
- **Scaling values, resource limits, alert thresholds, probe intervals** — gaps G-04/G-37 unchanged; no numbers manufactured.
- **CI/CD product mandates** — none; capability rules only.
