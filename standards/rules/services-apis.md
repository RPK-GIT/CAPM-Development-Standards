# CAP-SRV — Services & APIs

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md).

**Rules:** 9 active (0 Critical, 3 High, 6 Medium). All SAP references verified against official CAP documentation on **2026-08-11**.

Scope boundaries: input-validation annotations are governed by [CAP-SEC-012](security.md#cap-sec-012--validate-externally-writable-input-declaratively); service authorization by [CAP-SEC-001](security.md#cap-sec-001--model-authorization-explicitly-for-every-exposed-service); pagination limit values by CAP-SEC-014 and the future CAP-PERF category; handler mechanics by [CAP-LOGIC](business-logic.md).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-SRV-001 | Expose use-case projections, not persistence entities | High | SAP-REC | Both |
| CAP-SRV-002 | Rely on generic providers — do not reimplement framework behavior | High | SAP-REC | Both |
| CAP-SRV-003 | Prefer declarative techniques before imperative code | Medium | SAP-REC | Both |
| CAP-SRV-004 | Model custom operations as actions and functions with correct semantics | Medium | SAP-REQ | Both |
| CAP-SRV-005 | Serve new APIs with OData V4; OData V2 only as documented legacy | High | SAP-REQ | Both |
| CAP-SRV-006 | Make protocol exposure an explicit decision | Medium | SAP-REC | Both |
| CAP-SRV-007 | Use framework draft handling for draft requirements | Medium | SAP-REC | Both |
| CAP-SRV-008 | Enable optimistic concurrency for concurrently edited entities | Medium | SAP-REC | Both |
| CAP-SRV-009 | Serve media data through CAP's media handling | Medium | SAP-REC | Both |

---

## CAP-SRV-001 — Expose use-case projections, not persistence entities

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-001 |
| **Title** | Expose use-case projections, not persistence entities |
| **Category** | Services & APIs |
| **Severity** | High |
| **Authority** | SAP-REC |
| **Applicability** | Every externally served service entity |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-003 (service boundaries), CAP-SEC-001, CAP-SEC-011, CAP-SEC-012 (mass assignment) |
| **Last verified** | 2026-08-11 |

### Rule statement
Service entities MUST be deliberate projections tailored to the service's use case — selecting, excluding, or denormalizing elements as consumers need them — rather than 1:1 re-exposure of persistence entities. Every element and association a service exposes is a decision; internal, technical, and unrelated-use-case elements are excluded.

### Rationale
SAP: "Instead of exposing access to underlying data in a 1:1 fashion, services frequently expose denormalized views, tailored to specific use cases", and warns that exposing all entities 1:1 "open[s] huge entry doors to your clients with only few restrictions", forcing "numerous checks on a per-request basis" in "complex and expensive evaluations". Direct persistence exposure couples the API contract to the persistence model (every model change becomes an API change), leaks internal fields, and widens the writable surface. **High:** materially affects API compatibility and the exposure surface that CAP-SEC-011/-012 must then defend.

### Implementation guidance
- Start each service entity from the consumer's needs: which fields does *this* use case read or write? Project exactly those (`as projection on … excluding { … }` or explicit column lists).
- Persistence model, domain model, and API model do not have to be identical — projections are CAP's mechanism for keeping them decoupled.
- Fields that must never be client-written but must be read: keep them in the projection with `@readonly`; fields that must not even be read: exclude them.

### Evidence expected in code
`srv/**/*.cds` service entities defined as tailored projections (column selection/`excluding`/denormalization) with a recognizable relationship to the use case; no service that simply re-exposes the whole `db/` model.

### Detection guidance
1. Enumerate exposed service entities in `srv/**/*.cds` (skip `@protocol:'none'` services).
2. For each, compare the projection with the underlying `db/` entity: identical element list with no exclusions/selection = 1:1 exposure candidate.
3. Assess per service: a service exposing most domain entities 1:1 → FAIL (cite the service); individual deliberate 1:1 projections of simple entities (e.g., code lists) are acceptable — check for intent (annotations, naming, `@readonly`).
4. Check exposed elements for internal/technical fields (cost/margin fields, internal flags, foreign technical keys) reachable by the service's audience → FAIL per finding.
5. Cross-check associations/compositions in the projection against CAP-SEC-011's exposure review.
6. Report with file:line references per service entity.

### Good example
```cds
// srv/catalog-service.cds — read-only catalog view, internal fields excluded
service CatalogService {
  @readonly entity Books as projection on my.Books
    excluding { supplierCost, internalNotes, createdBy, modifiedBy };
}
```

### Bad example
```cds
// srv/catalog-service.cds — whole domain model re-exposed 1:1
service CatalogService {
  entity Books     as projection on my.Books;      // incl. supplierCost
  entity Suppliers as projection on my.Suppliers;  // unrelated to catalog use case
  entity Users     as projection on my.Users;      // leaks user administration
}
```

### Exception guidance
Deliberate full exposure of simple, non-sensitive entities (code lists, value-help entities) is legitimate — typically `@readonly` and often `@cds.autoexpose`d. Admin services may expose broad projections when the use case genuinely is administration; the breadth must match the audience and be stated in the service's documentation.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/providing-services (denormalized views tailored to use cases; 1:1 exposure warning)

---

## CAP-SRV-002 — Rely on generic providers — do not reimplement framework behavior

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-002 |
| **Title** | Rely on generic providers — do not reimplement framework behavior |
| **Category** | Services & APIs |
| **Severity** | High |
| **Authority** | SAP-REC |
| **Applicability** | All projects |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-002 (no custom frameworks), CAP-SRV-003, CAP-SRV-007, CAP-SRV-009 |
| **Last verified** | 2026-08-11 |

### Rule statement
Behavior that CAP's generic providers serve out of the box — CRUD on exposed entities, deep reads/writes along compositions, pagination and sorting, search, draft choreography, media streaming, input-validation and authorization enforcement — MUST NOT be reimplemented in custom handlers. Custom `on` handlers replace the generic ones for their event; they are written only where domain logic genuinely differs from generic behavior, and MUST NOT hand-code what the generic handler would have done.

### Rationale
SAP: "a service definition is all we need to run a full-fledged server out of the box. The need for coding reduces to real custom logic specific to a project's domain"; generic handlers "automatically serve all CRUD requests to entities" including "deeply nested document structures". Reimplementations are the primary source of framework drift: they silently lose enforced behavior (authorization checks, validation, pagination truncation, tenant isolation of the data access layer) and must be maintained forever. **High:** beyond technical debt, a hand-written CRUD handler that forgets what the generic one enforced materially affects data integrity and the security posture (cross-ref CAP-SEC-001/-012).

### Implementation guidance
- Before writing an `on` handler for a CRUD event, name the delta to generic behavior. No delta → no handler. Small delta → prefer `before`/`after` around the generic handler.
- In Node.js, custom `on` CRUD handlers that need generic behavior can delegate with `next()` rather than re-coding reads/writes.
- Actions/functions are the intended home for genuinely custom operations (CAP-SRV-004); in CAP Java they always need an `@On` handler — that is custom logic, not reimplementation.

### Evidence expected in code
Handler files containing only domain-specific logic; CRUD events without handlers (served generically) or with `before`/`after` enrichment; no hand-written paging/filtering/CRUD plumbing.

### Detection guidance
1. List all custom handlers (`srv/**/*.js|ts`: `srv.on/before/after`; Java: `@On/@Before/@After` methods) with their events and entities.
2. For each `on` handler on a CRUD event, determine the domain delta it implements. Handlers that re-code generic behavior (manual SELECT/INSERT mirroring the request, hand-built `$top/$skip` handling, manual deep-insert recursion, hand-rolled draft state machines) → FAIL with file:line.
3. Check `before`/`after` handlers stay enrichment/validation — an `after` handler re-querying and rebuilding the whole result set is a reimplementation signal.
4. Search for framework-duplicating utilities (custom pagination helpers, generic CRUD base classes) → FAIL (also cross-report CAP-ARCH-002).
5. For each FAIL, name the CAP-native capability that should serve the behavior.

### Good example
```js
// srv/catalog-service.js — generic CRUD untouched; only domain logic added
module.exports = srv => {
  srv.after('READ', 'Books', books =>
    books.forEach(b => { if (b.stock > 100) b.discount = '11%'; }));
};
```

### Bad example
```js
// re-implements generic READ incl. hand-built paging — loses implicit
// pagination, localization handling, and authorization-aware querying
srv.on('READ', 'Books', async req => {
  const { $top = 50, $skip = 0 } = req._queryOptions ?? {};
  return db.run(`SELECT * FROM my_Books LIMIT ${$top} OFFSET ${$skip}`); // also violates CAP-SEC-013
});
```

### Exception guidance
Legitimate full `on` replacements exist: virtual/computed result sets, remote-backed entities, results assembled from multiple sources. Document the reason at the handler and keep the replacement's behavior contract (pagination, localization where applicable) explicit — what the generic handler provided and this handler now must.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/served-ootb (generic providers; "coding reduces to real custom logic")
- https://cap.cloud.sap/docs/guides/services/custom-code (`on` handlers run instead of generic handlers)

---

## CAP-SRV-003 — Prefer declarative techniques before imperative code

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-003 |
| **Title** | Prefer declarative techniques before imperative code |
| **Category** | Services & APIs |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | All custom logic decisions in service definitions and implementations |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-002, CAP-SEC-010 (declarative instance auth), CAP-SEC-012 (declarative validation); absorbs candidate CAP-LOGIC #1 |
| **Last verified** | 2026-08-11 |

### Rule statement
Before implementing a requirement imperatively, developers (and Claude Code, per AI-DEV-004) MUST check whether CDS can express it declaratively — constraints (`@mandatory`, `@assert.*`), static restrictions (`@readonly`, `@insertonly`), authorization (`@requires`/`@restrict`), managed data, drafts, localized/temporal data, projections and calculated elements — and use the declarative form where it covers the requirement. Imperative handlers remain the correct choice for genuine domain behavior: cross-entity rules, orchestration, calculations, and integration logic that annotations cannot express. The choice, in borderline cases, is documented briefly at the handler.

### Rationale
SAP: "Consider first whether your custom logic can be addressed using declarative techniques, like status flows or declarative constraints, before resorting over to programmatic and hence imperative options." Declarative definitions are enforced uniformly by generic providers on every path (including drafts and future handlers), are visible in the model to reviewers and tools, and generate protocol metadata; equivalent handler code covers only the paths its author considered. This is deliberately **not** an absolute: imperative business logic is a first-class part of CAP — the rule targets *avoidable* imperative re-expressions of declarative capabilities. **Medium:** violations are maintainability/coverage defects; where they weaken enforcement they escalate through CAP-SEC-010/-012.

### Implementation guidance
- Rule of thumb: *facts about data* (required, format, range, immutable, restricted-to) → annotations; *behavior* (compute, orchestrate, integrate, decide) → handlers.
- When writing a `before` validation handler, first list which of its checks are single-element facts — move those to `@assert.*`/`@mandatory` and keep only the genuine cross-field/cross-entity logic in code.
- Keep the declarative parts close to the service definition so reviewers see the contract in one place.

### Evidence expected in code
Constraint/restriction annotations on service entities where requirements are declarative; handlers containing only logic that annotations cannot express.

### Detection guidance
1. Inspect `before`/`on` handlers performing validation or rejection (`req.error`/`req.reject`/`Messages`/`ServiceException` on input conditions).
2. Classify each check: single-element presence/format/range/enum/target-existence → expressible declaratively; flag each such check implemented imperatively while the element lacks the corresponding annotation → FAIL with file:line and the annotation that would replace it.
3. Inspect handlers rejecting operations wholesale (e.g., blocking UPDATE) where `@readonly`/`@insertonly` on the entity would express it → FAIL.
4. Do NOT flag imperative logic that annotations cannot express (cross-entity checks, computed decisions, remote validation) — that is compliant; note it as such when relevant.
5. Cross-report validation findings to CAP-SEC-012 where the input is security-relevant.

### Good example
```cds
// declarative facts in the model…
annotate AdminService.Books with {
  title @mandatory;
  stock @assert.range: [ 0, 99999 ];
};
```
```js
// …only genuine domain logic in code
srv.before('CREATE', 'Orders', async req => {
  const book = await SELECT.one.from(Books, req.data.book_ID, b => b.stock);
  if (book.stock < req.data.quantity) req.reject(409, 'ORDER_EXCEEDS_STOCK');
});
```

### Bad example
```js
// hand-coded re-expressions of @mandatory and @assert.range —
// enforced only on this path, invisible in the model and $metadata
srv.before(['CREATE','UPDATE'], 'Books', req => {
  if (!req.data.title) req.error(400, 'Title is required');
  if (req.data.stock < 0) req.error(400, 'Stock must be >= 0');
});
```

### Exception guidance
Dynamic rules (configurable at runtime), locale-/tenant-dependent constraints, and checks needing data access are legitimately imperative even when they resemble declarative ones — note the reason at the handler. Generated/legacy models that cannot be annotated may centralize validation imperatively with a documented pointer.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/custom-code ("Consider first whether your custom logic can be addressed using declarative techniques…")
- https://cap.cloud.sap/docs/guides/services/constraints (declarative constraints)
- https://cap.cloud.sap/docs/guides/services/providing-services (`@readonly`/`@insertonly`)

---

## CAP-SRV-004 — Model custom operations as actions and functions with correct semantics

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-004 |
| **Title** | Model custom operations as actions and functions with correct semantics |
| **Category** | Services & APIs |
| **Severity** | Medium |
| **Authority** | SAP-REQ (documented protocol semantics) |
| **Applicability** | All custom (non-CRUD) business operations |
| **Runtime** | Both (CAP Java: every action/function additionally needs a custom `@On` handler — no default exists) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-002, CAP-SRV-003, CAP-ARCH-002 (no bespoke HTTP endpoints) |
| **Last verified** | 2026-08-11 |

### Rule statement
Custom business operations MUST be modeled in CDS as actions or functions — never as misused CRUD events, magic filter values, or framework-bypassing HTTP routes. Semantics follow SAP's definition: "Actions modify data in the server. Functions retrieve data." — an operation with side effects is an action (served via POST), a pure read is a function (served via GET); data MUST NOT be modified in a function. Operations on a specific instance are bound (they receive the entity key implicitly); cross-entity operations are unbound or collection-bound.

### Rationale
Modeled operations are part of the API contract: typed parameters, protocol serving, authorization annotations (`@requires` on the operation), and metadata all come from the model. Modifying data through a GET-served function breaks HTTP semantics (caching, retries by intermediaries can silently repeat side effects) and evades protocol-level expectations. Hand-built express/Spring routes bypass CAP's request context entirely — authorization, transactions, tenant context (cross-ref CAP-ARCH-002). **Medium:** contract and semantics quality; escalates via the cross-referenced rules when auth/transactions are bypassed.

### Implementation guidance
- Name the operation for the business event (`submitOrder`, `cancel`), bind it to the entity it acts on, and give parameters CDS types (`@mandatory` where required).
- Node.js: implement via `srv.on('<operation>', …)`; Java: an `@On` handler is mandatory — CAP Java provides no default handler for actions/functions.
- Put `@requires`/`@restrict` on operations whose authorization differs from the service default (cross-ref CAP-SEC-001).

### Evidence expected in code
Custom operations declared in `srv/**/*.cds` (`action`/`function`, bound via `actions { }` blocks where instance-specific); no data modification inside functions; no side-channel operations (custom routes, overloaded CRUD).

### Detection guidance
1. Enumerate `action`/`function` declarations in `srv/**/*.cds` and their handlers.
2. For each `function`, inspect its handler for writes (INSERT/UPDATE/DELETE/emit of state-changing commands) → any write → FAIL (must be an action).
3. Search for business operations smuggled through CRUD: handlers branching on magic field values (`if (req.data.action === 'approve')` in UPDATE handlers) → FAIL (model as action).
4. Search for framework-bypassing routes (`app.post(` in server.js/custom middleware, `@RestController` endpoints touching business data) → FAIL here and under CAP-ARCH-002.
5. Java: verify every declared action/function has an `@On` handler (missing → runtime error; report as defect).
6. Report with file:line per operation.

### Good example
```cds
service OrderService {
  entity Orders as projection on my.Orders actions {
    @requires: 'OrderManager'
    action cancel(reason : String @mandatory);   // modifies → action/POST
  };
  function orderVolume(year : Integer) returns Decimal;  // pure read → GET
}
```

### Bad example
```cds
service OrderService {
  // "function" that cancels the order — side effect behind a GET
  function cancelOrder(ID : UUID) returns Boolean;
}
```
```js
// business operation smuggled through UPDATE with a magic field
srv.on('UPDATE', 'Orders', req => {
  if (req.data.__action === 'cancel') { /* … */ }
});
```

### Exception guidance
None on the modify-in-function prohibition. Technical endpoints genuinely outside the CDS model (webhook receivers for external systems) may live as custom routes only with a documented CAP-ARCH-002 exception.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/custom-actions ("Actions modify data in the server. Functions retrieve data."; bound operations receive the entity key)
- https://cap.cloud.sap/docs/java/cqn-services/application-services (CAP Java: no default `On` handlers for actions/functions)

---

## CAP-SRV-005 — Serve new APIs with OData V4; OData V2 only as documented legacy

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-005 |
| **Title** | Serve new APIs with OData V4; OData V2 only as documented legacy |
| **Category** | Services & APIs |
| **Severity** | High |
| **Authority** | SAP-REQ ("OData V2 is deprecated") |
| **Applicability** | All OData-served services |
| **Runtime** | Both (V2 mechanics differ: Node.js via community adapter `@cap-js-community/odata-v2-adapter`; Java V2 adapter built in — and served **by default**, see CAP-SRV-006) |
| **CAP version** | ⏱ Version-sensitive: deprecation stance and adapter packaging per current docs (verified on cds 10 / Java 5 era docs); re-verify at majors |
| **Status** | Active |
| **Related rules** | CAP-SRV-006, CAP-SEC-014 |
| **Last verified** | 2026-08-11 |

### Rule statement
New services and APIs MUST be served via OData V4 (CAP's default). OData V2 MAY be served only for SAP's documented reasons — existing UIs, or specific UI controls not yet working with V4 — and each V2 exposure MUST carry a documented justification naming that reason and a review date.

### Rationale
SAP: "OData V2 is deprecated. Use OData V2 only if you need to support existing UIs or if you need to use specific controls that don't work with V4 yet". CAP defaults to V4. Building new consumers on a deprecated protocol accrues compatibility debt against SAP's direction and third-party V2 tooling. **High:** materially affects API compatibility and long-term supportability of the contract.

### Evidence expected in code
V4 as the serving protocol for new services; where V2 is present: the adapter dependency (Node.js) or Java protocol config, plus a recorded justification (ADR/comment) naming the legacy UI/control.

### Detection guidance
1. Node.js: check for `@cap-js-community/odata-v2-adapter` in `package.json` and `@protocol` annotations mentioning `odata-v2`.
2. Java: check `@protocol`/`@protocols` annotations and adapter dependencies; note Java serves V2 **by default** — determine whether V2 exposure is deliberate or accidental (CAP-SRV-006).
3. For each V2-served service, locate the documented justification (existing UI / named control); missing → FAIL.
4. New services (created in this project's lifetime, no legacy consumers) served V2 → FAIL.
5. Report per service with file:line.

### Good example
```cds
// V2 only for the legacy tree-table UI, documented and scoped
@protocol: ['odata-v2']  // legacy: sap.ui.table.TreeTable in app/admin — see ADR-0012
service LegacyAdminService { /* … */ }
```

### Bad example
```text
Greenfield service consumed only by a new Fiori Elements V4 app,
but served via the V2 adapter "because the last project did it that way" —
no justification, deprecated protocol for a new contract.
```

### Exception guidance
The documented SAP reasons (existing UIs; controls not yet V4-capable) are the exception mechanism — recorded per service with a re-evaluation date. No other exceptions.

### SAP reference
- https://cap.cloud.sap/docs/guides/protocols/odata (V4 default; V2 deprecation and its only sanctioned uses; adapter packaging per runtime)

---

## CAP-SRV-006 — Make protocol exposure an explicit decision

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-006 |
| **Title** | Make protocol exposure an explicit decision |
| **Category** | Services & APIs |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented mechanisms and defaults) |
| **Applicability** | All services; particularly internal services and CAP Java projects |
| **Runtime** | Both — with runtime-specific defaults: Java serves `odata-v4` **and** `odata-v2` by default; Node.js defaults to OData V4 at `/odata/v4` |
| **CAP version** | Current documented defaults (verified 2026-08-11); re-verify at majors |
| **Status** | Active |
| **Related rules** | CAP-SEC-001, CAP-SRV-005, CAP-ARCH-003 |
| **Last verified** | 2026-08-11 |

### Rule statement
Which protocols (and paths) each service is served on MUST be a visible, deliberate decision: services meant for internal consumption only are annotated `@protocol: 'none'`; CAP Java projects MUST explicitly decide about the default V2+V4 double exposure (restrict with `@protocol`/`@protocols` where V2 is not wanted); `@path` values are relative unless single-protocol serving is intended (SAP: an absolute path "will disallow serving the service at multiple protocols").

### Rationale
SAP documents the mechanisms (`@protocol: 'none'` treats a service as internal; Java: "By default, a service is served by the protocols `odata-v4` and `odata-v2`"; the absolute-path constraint). Unmanaged exposure means services reachable over protocols nobody tests or secures deliberately — in Java, every service is a V2 endpoint unless someone decides otherwise. **Medium:** exposure *breadth* hygiene; actual unauthorized access is governed by CAP-SEC-001 (which still applies to every protocol).

### Implementation guidance
- Annotate service intent at definition time: internal → `@protocol: 'none'`; single-protocol → explicit `@protocol`; multi-protocol → list them.
- In Java, adopt a project default (e.g., `@protocol: 'odata-v4'` on all services, or global adapter configuration) so V2 exposure is opt-in, aligned with CAP-SRV-005.

### Evidence expected in code
`@protocol`/`@protocols` (or `@endpoints`) annotations expressing intent on services that deviate from "public V4 service"; internal services marked `none`; Java V2 stance visible in annotations or adapter configuration.

### Detection guidance
1. List services and their effective protocols: Node.js — `@protocol`/`@odata`/`@rest` annotations plus defaults (OData V4 at `/odata/v4`); Java — annotations plus the V4+V2 default.
2. Identify services consumed only service-to-service or by internal jobs (no UI/external consumer): lacking `@protocol: 'none'` → FAIL per service.
3. Java: services without protocol restriction → confirm V2 exposure is deliberate (justification per CAP-SRV-005); undecided → FAIL here.
4. Check `@path` values: absolute paths on services also annotated for multiple protocols → FAIL (documented conflict).
5. Report per service with file:line.

### Good example
```cds
@protocol: 'none'          // internal orchestration only — not served
service InternalRatingService { /* … */ }

@protocol: 'odata-v4'      // deliberate single-protocol exposure (Java: disables default V2)
service CatalogService { /* … */ }
```

### Bad example
```cds
// Java project: intended as internal helper, but unannotated —
// served publicly on BOTH /odata/v4 and /odata/v2 by default
service RecalculationHelper {
  action recalcAll();
}
```

### Exception guidance
Projects standardizing exposure globally (Java adapter config, Node.js `cds.env` protocol config) satisfy the rule at configuration level — the reviewer checks the config instead of per-service annotations.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-serve (`@protocol: 'none'`; absolute-path constraint; default endpoints)
- https://cap.cloud.sap/docs/java/cqn-services/application-services (default `odata-v4` + `odata-v2` serving; `@protocol`/`@endpoints`)

---

## CAP-SRV-007 — Use framework draft handling for draft requirements

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-007 |
| **Title** | Use framework draft handling for draft requirements |
| **Category** | Services & APIs |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | Services with draft/edit-session requirements (typically Fiori elements UIs); NOT APPLICABLE where no draft requirement exists |
| **Runtime** | Both |
| **CAP version** | All currently supported versions (new `@cap-js` database services require/auto-enable Lean Draft in Node.js) |
| **Status** | Active |
| **Related rules** | CAP-SRV-002, CAP-SEC-012 (validations must also cover active entities) |
| **Last verified** | 2026-08-11 |

### Rule statement
Where interrupted/resumable editing (drafts) is required, it MUST be implemented with CAP's draft support — `@odata.draft.enabled` on the service entity — not with hand-built draft tables, status flags, or client-side persistence. Validations MUST hold for active entities too, since SAP documents that "active entities can be updated directly, bypassing drafts" (enforced via CAP-SEC-012).

### Rationale
SAP: annotating with `@odata.draft.enabled` is "all you need to do"; "CAP automatically serves the Fiori draft choreography" — including the shadow draft entity, locking, and lifecycle events. A hand-built draft mechanism duplicates a large, subtle framework feature (CAP-SRV-002) and won't integrate with Fiori's draft protocol. **Medium:** framework-duplication/maintainability; the security-relevant validation caveat is owned by CAP-SEC-012.

### Evidence expected in code
`@odata.draft.enabled` on draft-requiring entities; no custom "draft"/"WIP" persistence structures duplicating the mechanism.

### Detection guidance
1. Determine whether draft requirements exist (Fiori elements apps in `app/`, requirements docs). None → NOT APPLICABLE.
2. Where they exist: check the relevant service entities for `@odata.draft.enabled`.
3. Search `db/` and `srv/` for hand-built draft mechanisms: entities named `*Draft*`/`*WIP*` mirroring business entities, status fields like `isDraft`, handlers persisting incomplete versions → FAIL with file:line.
4. Cross-check CAP-SEC-012: validations annotated/implemented so they apply to active-entity updates as well.
5. Report per entity.

### Good example
```cds
service AdminService {
  @odata.draft.enabled
  entity Books as projection on my.Books;
}
```

### Bad example
```cds
// hand-built draft mechanism duplicating framework behavior
entity BookDrafts : cuid { bookData : LargeString; owner : String; isComplete : Boolean; }
```

### Exception guidance
Non-UI "draft-like" domain states (e.g., a business `Quotation` that is a real domain object, not an edit session) are domain modeling, not drafts — not a violation. The distinction: edit-session persistence of an otherwise-final entity → framework drafts; genuine business lifecycle states → domain model.

### SAP reference
- https://cap.cloud.sap/docs/guides/uis/fiori (`@odata.draft.enabled`; "CAP handles everything else"; active entities updatable directly)

---

## CAP-SRV-008 — Enable optimistic concurrency for concurrently edited entities

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-008 |
| **Title** | Enable optimistic concurrency for concurrently edited entities |
| **Category** | Services & APIs |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | Exposed entities updated concurrently by multiple users/clients outside draft protection; NOT APPLICABLE for read-only or draft-locked entities |
| **Runtime** | Both (Java note: on ETag mismatch during custom-code updates, check `rowCount() == 0` — the runtime does not throw) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-007 (drafts lock instead), CAP-SEC-012 |
| **Last verified** | 2026-08-11 |

### Rule statement
Exposed entities subject to concurrent updates MUST have a deliberate concurrency-control decision; the CAP-native mechanism is `@odata.etag` on a suitable element (typically `modifiedAt`), causing conflicting updates to fail with **412 Precondition Failed** instead of silently overwriting ("lost update"). Where the decision is "last write wins", that MUST be recorded.

### Rationale
SAP documents ETag-based conflict detection as the annotation-level mechanism ("Enable ETags by adding the `@odata.etag` annotation to an element…"), with conflicts answered by 412. Without it, concurrent editors silently overwrite each other — a data-loss class defect invisible until users complain. **Medium:** data-quality risk scoped to genuinely concurrent entities; not a systemic integrity failure.

### Evidence expected in code
`@odata.etag` on `modifiedAt` (or equivalent) for concurrently edited entities; client/test handling of 412; or a recorded last-write-wins decision.

### Detection guidance
1. Identify exposed entities that are updatable and plausibly multi-user (from requirements, UI apps, role definitions); exclude `@readonly` and draft-enabled entities (drafts lock during editing).
2. For each, check for `@odata.etag` on an element (the entity typically includes `managed` for `modifiedAt`).
3. Neither annotation nor a documented concurrency decision → FAIL per entity.
4. Java projects with custom update code on ETagged entities: verify `rowCount()`-based conflict detection (mismatch does not throw) → missing check → FAIL.
5. Check a test covers the 412 conflict path (cross-check CAP-TEST evidence when that category is authored).

### Good example
```cds
entity Orders : cuid, managed { /* … */ }
annotate OrderService.Orders with @odata.etag: modifiedAt;
```

### Bad example
```text
Multi-user "Orders" maintenance UI on a plain updatable projection —
no @odata.etag, no draft, no recorded decision: concurrent edits
silently overwrite each other.
```

### Exception guidance
Last-write-wins is acceptable for low-stakes, high-churn data when recorded as a decision. Single-writer entities (system-maintained, single-role) are NOT APPLICABLE — state why in the review.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/served-ootb (`@odata.etag`; 412 conflict behavior)
- https://cap.cloud.sap/docs/java/working-with-cql/query-execution (ETag mismatch: check `rowCount() == 0`)

---

## CAP-SRV-009 — Serve media data through CAP's media handling

| Field | Value |
|---|---|
| **Rule ID** | CAP-SRV-009 |
| **Title** | Serve media data through CAP's media handling |
| **Category** | Services & APIs |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented mechanism; automatic streaming) |
| **Applicability** | Services storing or serving media/large binary content; NOT APPLICABLE otherwise |
| **Runtime** | Both |
| **CAP version** | All currently supported versions. Dev-parity caveat (documented): SQLite doesn't support streaming — LargeBinary is read whole into memory |
| **Status** | Active |
| **Related rules** | CAP-SRV-002; future CAP-PERF (memory behavior) and CAP-SEC-018/G-11 (upload scanning, ORG) |
| **Last verified** | 2026-08-11 |

### Rule statement
Media and large binary elements MUST be modeled with CAP's media-data annotations — `@Core.MediaType` (plus `@Core.IsMediaType`/`@Core.IsURL` variants where the type or content is externally referenced) — so the generic providers serve uploads and downloads via automatic streaming. Hand-built media endpoints, base64-in-JSON payload fields, and custom buffering layers MUST NOT be used where this mechanism covers the requirement.

### Rationale
SAP documents `@Core.MediaType` as the mechanism for media elements, with "the media data is streamed automatically" for read, create, and update. Unannotated large binaries or hand-built endpoints buffer whole payloads in memory and re-implement content-type negotiation the framework provides (CAP-SRV-002). The documented SQLite caveat (reads LargeBinary whole, in memory) is a dev/test-parity consideration, not a production pattern. **Medium:** resource behavior and framework duplication; escalates through CAP-PERF for measured memory issues.

### Evidence expected in code
`@Core.MediaType`-annotated media elements in the model; no custom upload/download routes; no `LargeBinary` elements returned as regular payload fields.

### Detection guidance
1. Search `db/**/*.cds` and `srv/**/*.cds` for `LargeBinary` elements: each must carry `@Core.MediaType` (directly or via projection annotation) → unannotated media element → FAIL.
2. Search for hand-built media endpoints (custom routes handling multipart/base64 upload, handlers reading whole streams into buffers) → FAIL with file:line (also CAP-ARCH-002/CAP-SRV-002 signal).
3. Check exposed projections don't include raw binary elements as normal fields consumed by list views.
4. Note (observation, not FAIL) test-parity implications where SQLite is the dev database and media sizes are large.

### Good example
```cds
entity Documents : cuid {
  content   : LargeBinary @Core.MediaType: mediaType;
  mediaType : String      @Core.IsMediaType;
}
```

### Bad example
```js
// custom express route buffering the whole upload — bypasses CAP's
// streaming, auth context, and content-type handling
app.post('/upload', async (req, res) => {
  const chunks = []; for await (const c of req) chunks.push(c);
  await INSERT.into(Documents).entries({ content: Buffer.concat(chunks).toString('base64') });
});
```

### Exception guidance
Externally stored content (object stores, attachment services/plugins) referenced via `@Core.IsURL` or a dedicated attachments plugin is compliant and often preferable for user-generated content — record the choice. Genuine protocol gaps (e.g., resumable-upload requirements beyond CAP's mechanism) justify custom endpoints with a documented CAP-ARCH-002 exception.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/media-data (`@Core.MediaType` variants; "the media data is streamed automatically"; SQLite caveat)
