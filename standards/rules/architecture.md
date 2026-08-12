# CAP-ARCH — Architecture & project structure

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md).

**Rules:** 7 active (0 Critical, 3 High, 4 Medium). All SAP references verified against official CAP documentation on **2026-08-11**. Note: SAP's former *Bad Practices* page (`/docs/about/bad-practices`) is **removed** from the current docs; rules previously grounded there are scoped to what remains citable on live pages (see CAP-ARCH-002).

Scope boundaries: service *projection* design is [CAP-SRV-001](services-apis.md); security exposure is `CAP-SEC`; deployment topology mechanics are [CAP-DEP](deployment.md).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-ARCH-001 | Use the standard project layout and framework conventions | Medium | SAP-REC | Both |
| CAP-ARCH-002 | No custom framework layers on top of CAP | High | SAP-REC | Both |
| CAP-ARCH-003 | Design services for single use cases | High | SAP-REC | Both |
| CAP-ARCH-004 | Services are stateless and process passive data | High | SAP-REC | Both |
| CAP-ARCH-005 | Keep application code platform- and protocol-agnostic | Medium | SAP-REC | Both |
| CAP-ARCH-006 | Start as a modulith; cut microservices late and deliberately | Medium | SAP-REC | Both |
| CAP-ARCH-007 | Record material architecture decisions | Medium | ORG | Both |

---

## CAP-ARCH-001 — Use the standard project layout and framework conventions

| Field | Value |
|---|---|
| **Rule ID** | CAP-ARCH-001 |
| **Title** | Use the standard project layout and framework conventions |
| **Category** | Architecture & project structure |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented defaults; "convention over configuration") |
| **Applicability** | All projects |
| **Runtime** | Both (the impl-file convention in item 3 is Node.js; Java uses `@ServiceName` classes under `srv/src/main/java`) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-007, CAP-CDS-007 |
| **Last verified** | 2026-08-11 |

### Rule statement
Projects MUST follow the `cds init` layout and the framework's zero-config conventions unless a documented reason exists: (1) domain models and database content in `db/`, service definitions and implementations in `srv/`, UI content in `app/`; (2) initial/sample data as CSV files in `db/data/` named `<namespace>-<Entity>.csv`; (3) Node.js service implementations in equally named `.js` files next to the service's `.cds` file (or the project's documented alternative registration). Deviations from defaults require configuration *and* a recorded reason.

### Rationale
SAP: "CAP uses defaults for many things you'd have to configure in other frameworks. The idea is that things just work out of the box, with zero configuration." The layout and naming conventions are what the toolchain (build, deploy, data loading, impl-file lookup) keys on — deviations cost configuration, surprise every CAP-literate reader, and silently disable zero-config behavior (e.g., misnamed CSVs simply don't load). **Medium:** maintainability and tooling friction; nothing breaks security or data.

### Implementation guidance
- Scaffold with `cds init` (and `cds add` facets) rather than hand-creating structure.
- When a monorepo or multi-module layout is genuinely needed, keep the db/srv/app split *within* each CAP module and record the layout decision (CAP-ARCH-007).

### Evidence expected in code
`db/`, `srv/`, `app/` folders in their conventional roles; `db/data/*.csv` following the naming pattern; Node.js impl files co-located and matching service definition names (or documented alternative).

### Detection guidance
1. Verify the top-level layout: models under `db/`, services under `srv/`, UI under `app/` (allowing documented monorepo variants).
2. List `db/data/` (and `test/data/`) CSVs; names not matching `<namespace>-<Entity>.csv` for existing entities → FAIL per file (they won't be loaded).
3. Node.js: for each `srv/*.cds` service with custom logic, verify the impl file convention (`.js` file of the same basename, or explicit `@impl`/documented registration).
4. Identify non-standard structure (models in `srv/`, handlers in random folders) → FAIL with locations unless a recorded reason exists (check ADRs/README per CAP-ARCH-007).
5. Report with file paths.

### Good example
```text
bookshop/
├── db/schema.cds
├── db/data/sap.capire.bookshop-Books.csv
├── srv/cat-service.cds
├── srv/cat-service.js          ← picked up automatically
└── app/
```

### Bad example
```text
bookshop/
├── model/entities.cds           ← models outside db/, needs config
├── db/data/books.csv            ← misnamed: silently not loaded
└── srv/handlers/logic.js        ← not found by convention, unregistered
```

### Exception guidance
Monorepos, multi-service projects, and reuse packages legitimately extend the layout — keep per-module conventions intact and record the structure decision. Generated projects from SAP tooling variants (e.g., Business Application Studio templates) are compliant by construction.

### SAP reference
- https://cap.cloud.sap/docs/get-started/ (layout; "zero configuration")
- https://cap.cloud.sap/docs/get-started/bookshop (CSV naming `db/data/<namespace>-<Entity>.csv`; impl files "placed next to a service definition's `.cds` file")

---

## CAP-ARCH-002 — No custom framework layers on top of CAP

| Field | Value |
|---|---|
| **Rule ID** | CAP-ARCH-002 |
| **Title** | No custom framework layers on top of CAP |
| **Category** | Architecture & project structure |
| **Severity** | High |
| **Authority** | SAP-REC (live concepts page criticizes DAO/DTO/Repository/Active-Records approaches and mandates determinations in event handlers; SAP's broader *Bad Practices* page is currently removed from the docs — see note) |
| **Applicability** | All projects |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-002, CAP-SRV-003, CAP-ARCH-004; AI guardrail AI-DEV-011/-012 |
| **Last verified** | 2026-08-11 (bad-practices page confirmed 404 with no successor URL) |

### Rule statement
Projects MUST NOT introduce framework-style abstraction layers between application code and CAP: no DAO/DTO/Repository layers or Active-Record-style data objects over CQL/CQN, no element-level validation/determination frameworks (validation and determinations belong in CDS annotations and event handlers), no code-generation layers producing per-entity boilerplate, and no wrapper facades re-exposing CAP APIs under project-specific names. Business logic talks to CAP's services and query APIs directly.

### Rationale
SAP's live core-concepts page explicitly criticizes "common DAOs, DTOs, Repositories, or Active Records approaches which use static classes" (conflicting with CAP's stateless services processing passive data) and states that validations and determinations go into event handlers, not data objects. Framework layers on CAP re-abstract an abstraction: they hide CAP capabilities (drafts, `$expand` push-down, instance-based authorization), decay as CAP evolves, and every team member must learn them on top of CAP. **High:** this is the single most expensive architectural mistake in CAP projects — it invalidates CAP-native-first across the codebase and routinely loses framework-enforced behavior on the way through the wrapper (data-integrity/security side effects via CAP-SRV-002/CAP-SEC rules).
*Documentation note:* SAP's dedicated *Bad Practices* page (which additionally named "abstracting from CAP" and code generation as anti-patterns) is removed from current docs (404, no successor found 2026-08-11); this rule is deliberately scoped to what live pages support, and the source map tracks the removal.

### Implementation guidance
- Handlers use `cds.ql`/CQL (Node.js) or the CQN query builders with static model classes (Java) directly — that *is* the data-access layer.
- Shared logic belongs in plain functions/services called from handlers, not in inheritance hierarchies or generic base handlers.
- If a repeated pattern feels framework-worthy, check for a CAP plugin/capability first (AI-DEV-004), then propose an ADR (CAP-ARCH-007) — never introduce it silently (AI-DEV-011).

### Evidence expected in code
Handlers using CAP APIs directly; absence of `*Repository`/`*DAO`/`*BaseHandler` hierarchies, entity-wrapper classes with behavior, per-element validator/determination registries, and generated boilerplate layers.

### Detection guidance
1. Search for layer indicators: class/file names matching `*Repository`, `*DAO`, `*Dao`, `*BaseService`, `*BaseHandler`, `*Mapper`+`*DTO` pairs in `srv/**`; generic per-entity CRUD helpers; decorator/registry systems dispatching per-element validators or determinations.
2. For each hit, determine whether it wraps CAP APIs (delegates to `cds.run`/`PersistenceService` underneath) → wrapper layer → FAIL with file:line.
3. Check for code generation producing per-entity artifacts beyond CAP's own tooling (`cds-typer`/CAP Java codegen are framework-native and fine) → project-specific generators emitting handler/DTO boilerplate → FAIL.
4. Distinguish compliant structuring: plain shared utility modules, domain services called from handlers, and typed model imports are NOT layers — do not flag them.
5. For each FAIL, name the CAP-native path (e.g., "use `SELECT.from(Books)` in the handler; delete `BooksRepository`").

### Good example
```js
// handler uses CAP's query API directly — no repository indirection
srv.before('CREATE', 'Orders', async req => {
  const { stock } = await SELECT.one.from(Books, req.data.book_ID, b => b.stock);
  if (stock < req.data.quantity) req.reject(409, 'ORDER_EXCEEDS_STOCK');
});
```

### Bad example
```java
// project-wide repository layer wrapping PersistenceService —
// hides CQN capabilities, static state conflicts with passive-data design
public class BookRepository extends BaseRepository<BookDto> {
  private static final Map<String, BookDto> cache = new HashMap<>(); // also CAP-MT-003!
  public BookDto findById(String id) { return mapToDto(db.run(...)); }
}
```

### Exception guidance
Thin, single-purpose adapters around *external* systems (a client for one legacy API) are integration code, not framework layers. A genuinely needed cross-cutting mechanism (e.g., a domain-event convention shared by many services) requires an approved ADR naming the CAP capability evaluated and why it was insufficient (CAP-ARCH-007, AI-DEV-011).

### SAP reference
- https://cap.cloud.sap/docs/get-started/concepts (criticism of DAO/DTO/Repository/Active-Records approaches; stateless services processing passive data; determinations in event handlers)
- Historic: https://cap.cloud.sap/docs/about/bad-practices — removed from current docs (tracked in [sap-cap-sources.md](../../references/sap-cap-sources.md))

---

## CAP-ARCH-003 — Design services for single use cases

| Field | Value |
|---|---|
| **Rule ID** | CAP-ARCH-003 |
| **Title** | Design services for single use cases |
| **Category** | Architecture & project structure |
| **Severity** | High |
| **Authority** | SAP-REC ("We strongly recommend…") |
| **Applicability** | All projects exposing services |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-001 (projection tailoring), CAP-SEC-001 (per-service authorization), CAP-SRV-006 |
| **Last verified** | 2026-08-11 |

### Rule statement
Each CDS service MUST serve one use case for one consumer group (e.g., `CatalogService` for browsing, `AdminService` for maintenance) — services are cheap to define. A single service exposing the whole domain model to all consumer groups is prohibited; conversely, the same domain entity MAY appear in several services, projected differently per use case.

### Rationale
SAP: "We strongly recommend designing your services for single use cases. Services in CAP are cheap, so there's no need to save on them." The documented anti-pattern — one service exposing everything — "open[s] huge entry doors to your clients with only few restrictions" and forces "numerous checks on a per-request basis" in complex, expensive evaluations. Use-case services make authorization trivially alignable with the audience (one `@requires` per service, CAP-SEC-001) instead of per-request compensations. **High:** service cut determines the exposure surface and the shape of the authorization model — a wrong cut materially complicates security and API compatibility, even though the direct violation is a design defect.

### Implementation guidance
- Cut services by *who* does *what*: browsing, administration, approval, integration — each gets its own service with its own restrictions.
- When one entity serves several audiences, project it per service (read-only slim view for browsing; richer updatable view for admins) — see CAP-SRV-001.

### Evidence expected in code
Multiple purpose-named services in `srv/`, each with audience-appropriate projections and service-level authorization; no monolithic "everything" service.

### Detection guidance
1. List services and their exposed entities (`srv/**/*.cds`).
2. Flag services exposing (nearly) the whole `db/` model to a broad audience: entity count ≈ domain entity count, mixed concerns (master data + transactions + configuration), single service consumed by multiple distinct roles → FAIL with the service name.
3. Check service naming/annotations reflect a use case and audience; verify each service's `@requires` matches one consumer group (cross-check CAP-SEC-001 results).
4. Confirm multi-audience entities are projected per service rather than shared through one service with per-request role branching in handlers (role-`if`s in handlers are a smell for a missing service cut).
5. Report per service with file:line.

### Good example
```cds
@requires: 'authenticated-user'
service CatalogService {           // use case: browse & order
  @readonly entity Books as projection on my.Books excluding { supplierCost };
}
@requires: 'admin'
service AdminService {             // use case: maintain catalog
  entity Books   as projection on my.Books;
  entity Authors as projection on my.Authors;
}
```

### Bad example
```cds
// one service, all entities, all audiences — authorization pushed into
// per-request handler checks
service EverythingService {
  entity Books; entity Authors; entity Orders; entity Users; entity Config;
}
```

### Exception guidance
Very small applications (single audience, few entities) may legitimately have exactly one service — that *is* the single use case; state it in the review. Technical services (integration facades) follow the same principle per integration partner.

### SAP reference
- https://cap.cloud.sap/docs/guides/services/providing-services ("strongly recommend … single use cases"; 1:1-exposure warning)

---

## CAP-ARCH-004 — Services are stateless and process passive data

| Field | Value |
|---|---|
| **Rule ID** | CAP-ARCH-004 |
| **Title** | Services are stateless and process passive data |
| **Category** | Architecture & project structure |
| **Severity** | High |
| **Authority** | SAP-REC (documented core design principle) |
| **Applicability** | All service implementations |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-002, CAP-MT-003 (tenant isolation — the multitenant escalation of this rule), CAP-MT-006 |
| **Last verified** | 2026-08-11 |

### Rule statement
Service implementations MUST be stateless: no conversational or per-user state held in instance/module/static variables between requests, and no behavior-carrying data objects (Active-Record-style entities with methods). Data is passive — plain objects (Node.js) / plain data maps and typed accessors (Java) — processed by handlers; any state lives in the database (or a deliberate, documented cache per CAP-MT-003 rules).

### Rationale
SAP's core concepts: "Services are **stateless** → process passive data"; "All data processed and served by CAP services is passive", with the static-class DAO/Active-Record criticism attached. In-process state breaks horizontal scaling (requests land on different instances), breaks restart/failover semantics, and — in multitenant apps — becomes cross-tenant leakage (CAP-MT-003, where it is Critical). **High:** production reliability under scale-out; escalates to Critical in multitenant contexts via CAP-MT-003.

### Implementation guidance
- Anything that must survive a request goes to the database; anything per-request rides on `req`/the event context.
- Node.js: no `let`/`var` at module scope holding request-derived data; Java: no mutable `static` fields in handler classes.
- Legitimate caches follow CAP-MT-003 (tenant-keyed, documented, evictable) — even in single-tenant apps, document them.

### Evidence expected in code
Handlers reading input from `req`/context and persisting via CAP services; no session/user state fields; entities as plain data.

### Detection guidance
1. Node.js: scan `srv/**/*.js|ts` for module-scope mutable bindings written inside handlers, and closures in `init()` capturing request data across invocations → FAIL with file:line.
2. Java: scan handler/service classes for non-final `static` fields or instance fields mutated per request (`@Component` singletons!) → FAIL.
3. Search for behavior-carrying data objects: entity classes with business methods mutating their own persisted state (Active-Record pattern) → FAIL (also CAP-ARCH-002).
4. Distinguish compliant constants/config (immutable, request-independent) — do not flag.
5. In multitenant projects, escalate confirmed findings to CAP-MT-003 (Critical) instead of reporting only here.

### Good example
```java
@Component @ServiceName("OrderService")
class OrderHandler implements EventHandler {
  @Before(event = CqnService.EVENT_CREATE, entity = Orders_.CDS_NAME)
  void check(CdsCreateEventContext ctx, List<Orders> orders) {
    orders.forEach(o -> { /* passive data in, decision out */ });
  }
}
```

### Bad example
```java
@Component @ServiceName("OrderService")
class OrderHandler implements EventHandler {
  private Orders lastOrder;                 // instance state in a singleton —
  private static int requestCount = 0;      // shared across all requests/users
}
```

### Exception guidance
Process-lifetime technical caches (compiled templates, static config) are fine when immutable or explicitly managed per CAP-MT-003's cache guidance. No exception for user/session/business state in process memory.

### SAP reference
- https://cap.cloud.sap/docs/get-started/concepts ("Services are stateless → process passive data"; passive data; static-class criticism)

---

## CAP-ARCH-005 — Keep application code platform- and protocol-agnostic

| Field | Value |
|---|---|
| **Rule ID** | CAP-ARCH-005 |
| **Title** | Keep application code platform- and protocol-agnostic |
| **Category** | Architecture & project structure |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented agnosticism principle) |
| **Applicability** | All application code (custom handlers, service consumption, configuration) |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-006, CAP-SRV-006, CAP-SEC-013 (raw SQL), CAP-DB-004/-005, CAP-INT-002/-007 |
| **Last verified** | 2026-08-11 |

### Rule statement
Application code MUST NOT hard-wire itself to specific protocols, databases, or platform services where CAP provides an agnostic abstraction: no protocol-specific request parsing in business logic (protocol adapters do that), no database-vendor SQL in handlers where CQL expresses the query, no direct broker/HTTP clients where CAP service consumption applies, and no environment-specific branching (`if (isCloudFoundry)`) in domain logic. Profile-based configuration carries environment differences.

### Rationale
SAP's core concepts: "Services are **agnostic** → platforms and protocols" — agnosticism is what enables CAP's inner-loop development (in-memory DBs, mocks, "airplane mode"), hybrid testing, and late deployment decisions (CAP-ARCH-006). Each hard-wired dependency removes one of those abilities for the whole project: HANA-only SQL kills local SQLite tests; protocol-specific handler code breaks the service on its other protocols. **Medium:** capability erosion and maintainability — concrete data-integrity/security consequences are owned by the cross-referenced rules.

### Implementation guidance
- Read inputs from the protocol-agnostic event context (`req.data`, typed event contexts) — never from raw HTTP artifacts in business logic.
- Keep environment differences in `cds.requires` profiles / Spring profiles, not in code branches.
- Where a vendor feature is genuinely needed (HANA-specific function), isolate it and record it (CAP-ARCH-007); the injection rules of CAP-SEC-013 apply to any native SQL.

### Evidence expected in code
Handlers using event contexts and CQL; profile-based config for environment differences; vendor-/protocol-specific code absent or isolated with a recorded reason.

### Detection guidance
1. Search handlers for protocol artifacts: `req._`/raw Express `req`/`res` usage, HTTP header parsing in business logic (Node.js); servlet/`HttpServletRequest` access in CAP Java handlers → FAIL per site.
2. Search for vendor SQL in handlers (raw `SELECT … FROM` strings with vendor functions) where CQL could express the query → FAIL (cross-check CAP-SEC-013 for injection aspects).
3. Search for environment branching in domain logic (`process.env.VCAP_*` checks, `if` on platform variables outside bootstrap/config) → FAIL.
4. Check environment differences are expressed via configuration profiles (`[production]` blocks, Spring profiles) → compliant pattern.
5. Report with file:line; isolated, ADR-documented vendor code → PASS with observation.

### Good example
```jsonc
// environment difference in configuration, not code
{ "cds": { "requires": {
    "db": { "kind": "sql" },                       // sqlite in dev
    "[production]": { "db": { "kind": "hana" } }
} } }
```

### Bad example
```js
srv.on('READ', 'Books', async req => {
  if (process.env.VCAP_APPLICATION) {                       // platform branching
    return db.run(`SELECT * FROM BOOKS WITH HINT(USE_HEX_PLAN)`); // HANA-only + raw SQL
  }
  return SELECT.from(Books);
});
```

### Exception guidance
Deliberate use of vendor-specific capabilities (HANA search/geo features, platform APIs without CAP abstraction) is legitimate when isolated behind one module and recorded via ADR (CAP-ARCH-007) — the exception covers the capability, not scattering it through handlers.

### SAP reference
- https://cap.cloud.sap/docs/get-started/concepts (services agnostic to platforms and protocols)
- https://cap.cloud.sap/docs/get-started/features (inner-loop development enabled by agnostic design)

---

## CAP-ARCH-006 — Start as a modulith; cut microservices late and deliberately

| Field | Value |
|---|---|
| **Rule ID** | CAP-ARCH-006 |
| **Title** | Start as a modulith; cut microservices late and deliberately |
| **Category** | Architecture & project structure |
| **Severity** | Medium |
| **Authority** | SAP-REC (explicit three-step recommendation) |
| **Applicability** | Deployment-unit decisions for CAP applications |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-005 (agnosticism is what keeps the cut cheap), CAP-ARCH-007 (the cut is ADR-worthy), CAP-DEP-001/-003 (deployment mechanics); absorbs candidate CAP-OPS #4 (topology by configuration) |
| **Last verified** | 2026-08-11 |

### Rule statement
New CAP applications SHOULD be built as a modulith — multiple CDS services, one deployment unit — following SAP's explicit guidance: avoid premature cuts into microservices; go for a modulith approach instead; cut into separate microservices later, only when really needed. Every cut into separate deployment units MUST have a recorded rationale (CAP-ARCH-007) naming the concrete need (independent scaling, isolation, team/release decoupling).

### Rationale
SAP, verbatim structure: "1. **Avoid** premature cuts into microservices → ends up in lots of pain without gains; 2. **Go for** a *modulith* approach instead; 3. Cut into separate microservices **later on** → only when you really need to." CAP's design (services as design-time modules, local/remote transparency) makes late cuts cheap — premature cuts buy distributed-system cost (latency, partial failure, deployment orchestration) without the need. **Medium:** an architecture-quality rule; a wrong cut is expensive but reversible and not a direct integrity/security failure.

### Implementation guidance
- Model bounded functionality as separate CDS *services* from day one (CAP-ARCH-003) — that keeps the future cut available without committing to it.
- When a cut is proposed, the ADR names the driver and the evidence (load profile, team topology), not "microservices are best practice".

### Evidence expected in code
Single deployment unit (one `mta.yaml` module set / chart) for young or small applications; where multiple CAP deployment units exist: an ADR per cut naming the driver.

### Detection guidance
1. Count CAP deployment units (server modules in `mta.yaml` / Helm chart, independent CAP apps in the repo/org for one product).
2. One unit → PASS.
3. Multiple units → locate the recorded rationale (ADR/architecture doc) for the cut(s); present and naming a concrete driver → PASS; absent → FAIL (undocumented distribution).
4. Look for distribution symptoms without documented need: shared database between "micro" services, synchronous call chains between the units for single user requests → report as observations supporting the FAIL.
5. NOT ASSESSABLE if deployment artifacts are outside the repository — name what's needed.

### Good example
```text
One product, one mta.yaml: srv (all CDS services), db, app — modulith.
ADR-0009 (2026-05): order-processing split into its own deployment unit
after sustained queue-depth evidence; async-only coupling via events.
```

### Bad example
```text
Greenfield app started as five CAP "microservices" sharing one HDI
container, calling each other synchronously per request — no recorded
driver for any cut.
```

### Exception guidance
Genuine day-one drivers exist (regulatory isolation, radically different scaling profiles, separate teams with separate release trains) — the exception *is* the recorded ADR; nothing else is needed. Existing distributed landscapes are assessed against the ADR requirement only, not forced to merge.

### SAP reference
- https://cap.cloud.sap/docs/get-started/features (avoid premature cuts; modulith; cut later only when really needed)

---

## CAP-ARCH-007 — Record material architecture decisions

| Field | Value |
|---|---|
| **Rule ID** | CAP-ARCH-007 |
| **Title** | Record material architecture decisions |
| **Category** | Architecture & project structure |
| **Severity** | Medium |
| **Authority** | ORG (organizational policy; aligned with SAP's convention-over-configuration stance — deviations are what need records) |
| **Applicability** | All projects; triggered by the decision types listed in the statement |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | AI-DOC-001/-002 (AI-side duty), CAP-ARCH-001/-002/-005/-006 (rules whose exceptions require these records), CAP-SRV-005 |
| **Last verified** | 2026-08-11 (ORG policy — no SAP claim to verify) |

### Rule statement
The following decision types MUST be recorded as an ADR (or equivalent versioned record in the repository): major service-boundary and deployment-unit cuts (CAP-ARCH-006); introduction of any cross-cutting mechanism or framework-like construct (CAP-ARCH-002 exceptions); persistence-architecture deviations (non-default databases, external/native database access, shared persistence); major integration architecture (new external systems, messaging topology); deviations from CAP defaults and from rules of this standard (exception records per AI-DOC-002); deprecated-protocol exposure (CAP-SRV-005). Routine implementation choices do NOT require ADRs.

### Rationale
Architecture decisions outlive their authors; in a convention-over-configuration framework, *deviations from convention* are precisely the knowledge that is invisible in code. Reviews against this standard (and future maintainers) need the recorded "why" to distinguish a considered exception from drift — several rules in this catalog explicitly key their exceptions to such records. **Medium:** a process/traceability control. **ORG authority:** SAP does not prescribe ADRs; this is our policy, deliberately scoped to material decisions to avoid ADR inflation.

### Implementation guidance
- Keep records lightweight: context, decision, alternatives considered (naming the CAP-native option evaluated), consequences — one page, versioned with the code (e.g., `docs/adr/NNNN-*.md`).
- Write the ADR when the decision is made, not retroactively at review time.

### Evidence expected in code
A discoverable decision log (e.g., `docs/adr/`) containing records for every triggering decision present in the codebase.

### Detection guidance
1. Locate the decision log (`docs/adr/`, `docs/decisions/`, architecture section in README) — none while triggering decisions exist → FAIL.
2. From the review of other rules, collect triggering facts actually present: multiple deployment units, custom cross-cutting mechanisms, non-default persistence, V2 exposure, documented standard exceptions.
3. For each triggering fact, locate its record → missing → FAIL per decision (cite the code evidence and the absent record).
4. Spot-check records are substantive (context + alternatives + consequence), not placeholders.
5. Do NOT flag ordinary implementation choices lacking ADRs — scope is the statement's list.

### Good example
```text
docs/adr/0007-postgresql-for-edge-deployment.md
  Context: offline edge sites, no HANA Cloud reachability …
  Decision: @cap-js/postgres for edge tier; HANA remains default …
  Alternatives: SQLite (rejected: concurrency), HANA replication (cost) …
  Consequences: no MT support on edge tier; upgrade testing doubled …
```

### Bad example
```text
Repository contains a second deployment unit, a custom "validation
registry", and OData V2 exposure — and no decision record anywhere
explaining any of them.
```

### Exception guidance
Teams with an established equivalent mechanism (RFC process, architecture wiki with versioned exports referenced from the repo) satisfy the rule — the record must be discoverable from the repository. Prototypes/spikes explicitly marked non-production are exempt until productization.

### SAP reference
None (authority: ORG). Related SAP reading: https://cap.cloud.sap/docs/get-started/ (convention over configuration — the baseline whose deviations this rule documents).
