# CAP-LOGIC — Business logic & event handlers

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md).

**Rules:** 5 active (0 Critical, 0 High, 5 Medium). All SAP references verified against official CAP documentation on **2026-08-11**.

Scope boundaries — this category deliberately does **not** restate: declarative-before-imperative ([CAP-SRV-003](services-apis.md)), generic-provider reliance ([CAP-SRV-002](services-apis.md)), action/function semantics incl. the Java `@On`-handler requirement ([CAP-SRV-004](services-apis.md)), transaction participation and await-discipline ([CAP-TXN-001/-006](transactions.md)), error-response mechanics ([CAP-ERR](error-handling.md)), tenant/user context in async work ([CAP-MT-006](multitenancy.md), [CAP-EVT-004](events-messaging.md)), and statelessness ([CAP-ARCH-004](architecture.md)). These rules govern *where business logic lives and how handlers are built*.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-LOGIC-001 | Implement domain behavior in event handlers on the owning service | Medium | SAP-REC | Both |
| CAP-LOGIC-002 | Use handler phases according to their documented semantics | Medium | SAP-REC | Both |
| CAP-LOGIC-003 | Complete Node.js service implementations correctly | Medium | SAP-REQ | Node.js |
| CAP-LOGIC-004 | Register Java handlers the documented way, with typed event contexts | Medium | SAP-REC | Java |
| CAP-LOGIC-005 | Register Java logic on the right service layer | Medium | SAP-REC | Java |

---

## CAP-LOGIC-001 — Implement domain behavior in event handlers on the owning service

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOGIC-001 |
| **Title** | Implement domain behavior in event handlers on the owning service |
| **Category** | Business logic & event handlers |
| **Severity** | Medium |
| **Authority** | SAP-REC (event handlers are the documented custom-logic mechanism; determinations/validations belong in handlers per the concepts page — no verbatim "must live in handlers" sentence exists) |
| **Applicability** | All custom business logic |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-002/-003 (what to implement at all), CAP-ARCH-002 (no framework layers), CAP-ARCH-004 (passive data), CAP-SRV-004 (operations), CAP-LOGIC-005 (which Java service) |
| **Last verified** | 2026-08-11 |

### Rule statement
Custom domain behavior — validations, determinations, calculations, orchestration — MUST be implemented as event handlers registered on the CDS service that owns the behavior (Node.js: service implementations registering `before`/`on`/`after`; Java: handler classes per CAP-LOGIC-004). Business rules MUST NOT live in places the framework cannot see or govern: UI layers, database procedures/triggers holding domain rules, detached scripts writing to the database directly, or behavior-carrying data objects (CAP-ARCH-004).

### Rationale
Event handlers are the single custom-logic mechanism CAP's documentation presents ("Within your custom implementations, you can register event handlers"), and SAP's concepts page places validations and determinations "into event handlers", not data objects. Logic in handlers automatically participates in the request's transaction (CAP-TXN-001), tenant context, and error semantics — logic elsewhere silently bypasses all three and gets missed by every path that doesn't go through it (a validation in the UI is absent for API clients; a rule in a DB trigger is invisible to reviews and inconsistent with drafts). **Medium:** placement quality with escalation paths — bypassed validation escalates via CAP-SEC-012, bypassed tenancy via CAP-MT-003.

### Implementation guidance
- Ask per rule: *which service event does this behavior belong to?* Register it there (`srv.before('CREATE', 'Orders', …)`), keeping the handler thin and delegating to plain functions/classes for testability.
- Reused logic across services stays in plain modules *called from* handlers — reuse of functions, not relocation of the trigger point (and no framework layers per CAP-ARCH-002).
- UI-side checks are UX sugar only — the authoritative rule always exists server-side in a handler or annotation (CAP-SRV-003 decides which).

### Evidence expected in code
Domain rules discoverable in service implementations (`srv/**` handlers); no business rules embedded solely in `app/` UI code, `.hdbprocedure`/trigger artifacts, or standalone DB-writing scripts.

### Detection guidance
1. Inventory domain rules from requirements/tests (validations, calculations, status transitions).
2. For each, locate the implementation: service handler or CDS annotation → PASS; found only in UI code (`app/**` validation logic), database procedures/triggers, or external scripts → FAIL with file:line.
3. Search `db/src/**` for `.hdbprocedure`/trigger artifacts containing business conditions (not just technical DDL) → each → FAIL unless covered by a documented exception.
4. Search for direct-to-database write paths bypassing services (scripts using DB credentials, jobs calling the database driver) → FAIL here (and cross-report CAP-DB-004/CAP-MT-003 as applicable).
5. Report per rule/artifact.

### Good example
```js
// srv/order-service.js — the discount rule lives on the owning service event
const { computeDiscount } = require('./lib/pricing');
module.exports = srv => {
  srv.before('CREATE', 'Orders', req => {
    req.data.discount = computeDiscount(req.data);   // thin handler, testable lib
  });
};
```

### Bad example
```sql
-- db/src/order_discount.hdbprocedure — domain rule hidden in the database:
-- invisible to CAP (drafts, validations, events), unreviewable in the model
CREATE PROCEDURE apply_discount AS BEGIN
  UPDATE ORDERS SET DISCOUNT = 0.11 WHERE STOCK > 100;
END
```

### Exception guidance
Documented performance-motivated database logic (mass calculations pushed down per future CAP-PERF guidance, HANA procedures via CAP-DB-004's native-SQL exception) is legitimate when the *rule of record* stays visible (ADR per CAP-ARCH-007) and handlers remain the trigger point. UI duplication *in addition to* server-side enforcement is fine.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/custom-code (event handlers as the custom-logic mechanism)
- https://cap.cloud.sap/docs/get-started/concepts (validations/determinations in event handlers, not data objects)

---

## CAP-LOGIC-002 — Use handler phases according to their documented semantics

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOGIC-002 |
| **Title** | Use handler phases according to their documented semantics |
| **Category** | Business logic & event handlers |
| **Severity** | Medium |
| **Authority** | SAP-REC (phase mechanics are documented facts; the placement guidance derives from them) |
| **Applicability** | All custom event handlers |
| **Runtime** | Both — with Node.js-specific concurrency semantics stated below (Java ordering is explicit via `@HandlerOrder`, CAP-LOGIC-004) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-002 (`on` replaces generic handlers), CAP-SRV-003, CAP-SEC-012 (validation coverage), CAP-TXN-006; absorbs candidate CAP-LOGIC #4 (error collection) |
| **Last verified** | 2026-08-11 |

### Rule statement
Handlers MUST be placed in the phase whose documented mechanics match their purpose: `before` handlers run before the `on` handlers — input validation and input-mutating preparation belong here, collecting all input errors via `req.error()` (which "collects errors in `req.errors`" for a combined response) and using `req.reject()` only for immediate aborts; `on` handlers "run instead of the generic/default handlers" — only genuine replacements/implementations belong here (CAP-SRV-002); `after` handlers "run after the `on` handlers, and get the result set as input" — result enrichment belongs here, never primary validation of input (too late: the operation has executed). **Node.js concurrency MUST be respected:** `before` and `after` handlers "are always executed concurrently" — multiple handlers of the same phase MUST NOT depend on each other's effects or execution order; `on` handlers run sequentially as an interceptor chain (`next()`) **for requests**, but **concurrently for asynchronous events** — event `on` handlers must not assume chaining.

### Rationale
Phase misuse produces timing defects: validation in `after` runs post-execution (the write already happened); order-dependent `before` handlers race (documented concurrent execution); an event `on` handler calling `next()` per request semantics misbehaves under the documented concurrent event dispatch. These are subtle, load-dependent bugs. **Medium:** correctness-of-design guidance whose worst instances (missed validation) escalate through CAP-SEC-012; the mechanics are loud in tests only when tests cover concurrency.

### Implementation guidance
- One phase question per handler: *prepare/veto* → `before`; *replace how it's done* → `on`; *decorate the result* → `after`.
- Never share mutable state between same-phase handlers (they run concurrently in Node.js); if two steps must be ordered, make them one handler.
- Collect validation failures with `req.error()` so consumers get all problems at once; reserve `req.reject()` for conditions where continuing is pointless.

### Evidence expected in code
Validation/preparation in `before`, enrichment in `after`, `on` only for genuine implementations/replacements; no inter-handler ordering assumptions within a phase; `req.error` used for multi-error validation.

### Detection guidance
1. List all handler registrations with phase, event, and entity (`srv.before/on/after`; Java `@Before/@On/@After` — Java ordering assessed under CAP-LOGIC-004).
2. Flag input validation performed in `after` handlers (checks on `req.data` with error/reject after execution) → FAIL (too late).
3. Flag `before`/`after` handlers on the same event+entity where one reads state the other writes (order dependency under documented concurrency) → FAIL with both file:line sites.
4. Flag `on` handlers registered for asynchronous *events* that call `next()` or assume chain position → FAIL (request-only semantics).
5. Check multi-condition validation uses `req.error` collection rather than rejecting on the first finding (observation if single-check; FAIL only where UIs demonstrably need combined errors and get serial rejects).
6. Report per handler.

### Good example
```js
srv.before('CREATE', 'Orders', req => {                 // validate & prepare
  if (!req.data.items?.length) req.error(400, 'ORDER_EMPTY', 'items');
  if (req.data.quantity > 1000) req.error(400, 'QUANTITY_LIMIT', 'quantity');
});
srv.after('READ', 'Orders', orders =>                    // enrich results
  orders.forEach(o => o.isRush = o.deliveryDays <= 2));
```

### Bad example
```js
// validation in AFTER — the insert already happened; and two BEFORE
// handlers with an ordering dependency race (documented concurrency)
srv.after('CREATE', 'Orders', (order, req) => {
  if (!order.items?.length) req.reject(400, 'ORDER_EMPTY');
});
srv.before('CREATE', 'Orders', req => { req.data.total = sum(req.data.items); });
srv.before('CREATE', 'Orders', req => { req.data.tax = req.data.total * 0.19; }); // total may not exist yet
```

### Exception guidance
Post-execution *consistency checks* that intentionally veto after the fact (Java `After` handlers may abort by throwing; changeset-level vetoes via CAP-TXN-004's `beforeClose`) are legitimate when the check genuinely needs the result — distinguish from misplaced input validation and say so at the handler.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/custom-code (phase mechanics: before / instead-of / after-with-result)
- https://cap.cloud.sap/docs/node.js/core-services (`before`/`after` "always executed concurrently"; `on` sequential for requests, concurrent for events)
- https://cap.cloud.sap/docs/node.js/events (`req.error` collects into `req.errors`; `req.reject` throws immediately)

---

## CAP-LOGIC-003 — Complete Node.js service implementations correctly

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOGIC-003 |
| **Title** | Complete Node.js service implementations correctly |
| **Category** | Business logic & event handlers |
| **Severity** | Medium |
| **Authority** | SAP-REQ (`super.init()` call required); the impl-file convention is SAP-REC ("easiest way") |
| **Applicability** | Node.js projects with custom service implementations |
| **Runtime** | Node.js (Java counterpart: CAP-LOGIC-004) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-001 (impl-file location convention), CAP-SRV-002 |
| **Last verified** | 2026-08-11 |

### Rule statement
Node.js service implementations subclassing `cds.ApplicationService`/`cds.Service` and overriding `init()` MUST call `super.init()` — SAP: "Ensure to call `super.init()` to allow subclasses to register their handlers" — and MUST place the call deliberately: registrations made *before* the `super.init()` call run before those of subclasses, registrations after it run after (documented ordering lever; the conventional form is a final `return super.init()`). Implementations SHOULD be registered via the documented conventions (same-named `.js` file next to the `.cds` file — the documented "easiest way" — or the explicit `@impl` annotation / `impl` configuration), so the runtime actually picks them up.

### Rationale
A missing `super.init()` silently drops the framework's and superclasses' handler registrations — generic CRUD serving degrades or disappears with no error message, presenting as "CAP mysteriously stopped serving my entities". An unpicked-up impl file (misnamed, wrong folder, no registration) is the same failure by another route: the service serves purely generically and every custom rule silently doesn't exist. **Medium:** functionally loud once any test touches the affected behavior, but notoriously confusing to diagnose; the correction from the Phase 1 inventory (High → Medium) reflects that tests catch it.

### Evidence expected in code
Every overridden `init()` calling `super.init()` (typically `return super.init()` last); impl files following the naming convention or explicit registration; custom handlers demonstrably active (tests exercising them).

### Detection guidance
1. Search `srv/**/*.js|ts` for classes extending `cds.ApplicationService`/`cds.Service` with an `init()` method.
2. `init()` without a `super.init()` call → FAIL with file:line.
3. `super.init()` present but not returned/awaited → FAIL (the promise must be part of the lifecycle).
4. For each service definition with custom logic: verify the impl is registered (same-named file per CAP-ARCH-001, `@impl` annotation, or `impl` config) — an impl file that matches nothing → FAIL (dead handlers).
5. Cross-check at least one test exercises a custom handler per service (proves registration end-to-end); absent → missing-evidence note.

### Good example
```js
// srv/cat-service.js — next to srv/cat-service.cds
const cds = require('@sap/cds');
class CatalogService extends cds.ApplicationService {
  init() {
    this.before('CREATE', 'Orders', validateOrder);
    return super.init();   // deliberate: our handlers registered first
  }
}
module.exports = CatalogService;
```

### Bad example
```js
class CatalogService extends cds.ApplicationService {
  init() {
    this.before('CREATE', 'Orders', validateOrder);
    // super.init() missing — framework/generic registrations silently dropped
  }
}
```

### Exception guidance
The plain functional style (`module.exports = srv => { srv.before(...) }`) has no `init()` and is equally compliant — this rule binds the subclassing style. Deliberate pre-`super.init()` registration for ordering is the documented mechanism, not an exception.

### SAP reference
- https://cap.cloud.sap/docs/node.js/core-services ("Ensure to call `super.init()`…"; ordering via placement; impl-file conventions, `@impl`)

---

## CAP-LOGIC-004 — Register Java handlers the documented way, with typed event contexts

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOGIC-004 |
| **Title** | Register Java handlers the documented way, with typed event contexts |
| **Category** | Business logic & event handlers |
| **Severity** | Medium |
| **Authority** | SAP-REC (the `EventHandler` interface "is required" — REQ element; typed contexts "whenever possible", async-completion "not recommended" — REC) |
| **Applicability** | CAP Java projects with custom handlers |
| **Runtime** | Java (Node.js counterpart: CAP-LOGIC-003) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-LOGIC-002 (phase placement), CAP-LOGIC-005, CAP-SRV-004 (actions need `@On` handlers) |
| **Last verified** | 2026-08-11 |

### Rule statement
CAP Java handlers MUST follow the documented registration pattern: Spring beans (`@Component`) implementing the `EventHandler` marker interface (SAP: "required for CAP to identify the class"), scoped via `@ServiceName` (class-level default or per method), with methods annotated `@Before`/`@On`/`@After`. Handlers SHOULD use the event-specific type-safe Event Context interfaces (SAP: "whenever possible") rather than generic contexts; `@On` handlers for synchronous events complete via `setResult(...)`/`setCompleted()`; ordering needs within a phase are expressed with `@HandlerOrder`, never by registration accident; and `@On` handlers for **asynchronous events** SHOULD NOT complete the event (SAP: "not recommended… other handlers might not get notified").

### Rationale
The registration pattern is what makes handlers discoverable to CAP's scanning — a handler class missing `EventHandler` is silently never called (the Spring bean exists; CAP ignores it). Typed contexts move model drift to compile time; generic contexts with string keys fail at runtime. Completing async events in `@On` suppresses sibling handlers — the Java mirror of Node's request/event distinction (CAP-LOGIC-002). **Medium:** mis-registration is silent but surfaces in any behavioral test; the rest is compile-time-safety and correctness hygiene.

### Evidence expected in code
Handler classes with `@Component` + `EventHandler` + `@ServiceName` and phase annotations; typed contexts (e.g., `CdsCreateEventContext`, generated `*Context` types for actions); `@HandlerOrder` where ordering matters; no `setCompleted()` in async-event `@On` handlers.

### Detection guidance
1. List classes with `@Before/@On/@After` methods in `srv/src/main/java/**`.
2. Any such class not implementing `EventHandler` or not a Spring bean → FAIL (handler never registered) with file:line.
3. Methods using generic `EventContext` + string-based data access where an event-specific typed context exists → observation; systematic → FAIL (Medium).
4. Handlers whose correctness depends on execution order without `@HandlerOrder` (e.g., two `@Before` methods where one consumes the other's output) → FAIL.
5. `@On` handlers for async events calling `setCompleted()` → FAIL (documented "not recommended", suppresses other consumers).
6. Cross-check every modeled action/function has its `@On` handler under CAP-SRV-004's detection (do not re-report here).

### Good example
```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
class OrderHandler implements EventHandler {
  @Before(event = CqnService.EVENT_CREATE, entity = Orders_.CDS_NAME)
  void validate(CdsCreateEventContext ctx, List<Orders> orders) {   // typed context
    orders.forEach(o -> { if (o.getItems().isEmpty())
      throw new ServiceException(ErrorStatuses.BAD_REQUEST, "ORDER_EMPTY"); });
  }
}
```

### Bad example
```java
@Component   // EventHandler interface missing — CAP never finds this class;
class OrderHandler {                       // handlers silently inactive
  @Before(event = "CREATE", entity = "my.Orders")
  void validate(EventContext ctx) {
    Map<String,Object> data = (Map) ctx.get("data");   // stringly-typed access
  }
}
```

### Exception guidance
Generic contexts are legitimate in genuinely generic components (cross-entity auditing utilities) — SAP's "whenever possible" is the escape hatch; note the purpose at the class. Programmatic handler registration (documented advanced API) is compliant where the annotation style can't express the need.

### SAP reference
- https://cap.cloud.sap/docs/java/event-handlers/ (registration pattern; `EventHandler` "required"; typed contexts "whenever possible"; `setResult`/`setCompleted`; `@HandlerOrder`; async completion "not recommended")

---

## CAP-LOGIC-005 — Register Java logic on the right service layer

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOGIC-005 |
| **Title** | Register Java logic on the right service layer |
| **Category** | Business logic & event handlers |
| **Severity** | Medium |
| **Authority** | SAP-REC ("Event handlers implementing business or domain logic should be registered on an Application Service") |
| **Applicability** | CAP Java projects with custom handlers |
| **Runtime** | Java (no Node.js counterpart rule: the Node model registers on the application service by construction; DB-service handlers exist but follow the same principle via this rule's rationale) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-LOGIC-001, CAP-LOGIC-004, CAP-DB-004 |
| **Last verified** | 2026-08-11 |

### Rule statement
Business/domain logic MUST be registered on the Application Service (SAP: handlers "implementing business or domain logic should be registered on an Application Service"); Persistence Service handlers are reserved for technical, cross-cutting concerns (SAP's example: "triggering some code whenever an entity is written to the database" — e.g., technical auditing, replication hooks). Domain rules MUST NOT be attached to the Persistence Service to catch "all writes" as a shortcut.

### Rationale
Application Service handlers see the request's business context (protocol-level event, draft state, the service's projection of the data); Persistence Service handlers fire for *every* writer — including internal technical writes, draft persistence, and other services sharing entities — with the persistence shape, not the service projection. A domain validation attached there fires in contexts it was never designed for (breaking drafts or technical jobs) or silently sees different field sets than the author assumed. **Medium:** wrong-layer logic produces context-dependent defects; the documented split makes intent reviewable.

### Evidence expected in code
Domain rules in handlers scoped to application services (`@ServiceName("<AppService>")`); Persistence Service handlers (`@ServiceName(PersistenceService.DEFAULT_NAME)`) containing only technical concerns.

### Detection guidance
1. List handlers scoped to the Persistence Service.
2. Classify each: technical/cross-cutting (audit hook, replication, technical field stamping) → PASS; domain conditions (business validations, status rules, calculations feeding business fields) → FAIL with file:line and the owning application service as the target.
3. Check domain handlers on application services aren't duplicated on the persistence layer "for safety" (double execution) → FAIL the duplicate.
4. Report per handler.

### Good example
```java
@Component @ServiceName(AdminService_.CDS_NAME)          // domain rule: app service
class BookValidation implements EventHandler {
  @Before(event = CqnService.EVENT_CREATE, entity = Books_.CDS_NAME)
  void checkTitle(List<Books> books) { /* business validation */ }
}
```

### Bad example
```java
@Component @ServiceName(PersistenceService.DEFAULT_NAME) // domain rule on the DB
class BookValidation implements EventHandler {           // service: fires for drafts,
  @Before(event = CqnService.EVENT_CREATE, entity = Books_.CDS_NAME)
  void checkTitle(List<Books> books) { /* … */ }          // technical writes, every
}                                                          // other service — contexts
                                                           // it was never designed for
```

### Exception guidance
Genuinely universal technical invariants (e.g., stamping a technical checksum on every write) belong on the Persistence Service — that is the documented purpose, not an exception. A domain rule that truly must bind all writers regardless of entry point should first check whether the model can express it (CAP-SRV-003) before Persistence Service placement, and record the decision.

### SAP reference
- https://cap.cloud.sap/docs/java/cqn-services/application-services (business logic on Application Services "should"; Persistence Service for technical requirements)
