# CAP-DB — Data access & persistence

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-24, G-25, G-26 in [research-gaps.md](../../references/research-gaps.md).

**Rules:** 10 active (0 Critical, 4 High, 5 Medium, 1 Low). All SAP references verified against official CAP documentation on **2026-08-11**.

Scope boundaries: **injection safety is [CAP-SEC-013](security.md#cap-sec-013--construct-queries-injection-safe)** — the rules here reference it and do not restate it. Model-level persistence design (keys, compositions, managed data) is `CAP-CDS`; pagination values and query-performance modeling are [CAP-PERF](performance.md) (limits decisions: CAP-SEC-014); deploy formats (`hdbtable` vs `hdbcds`) are [CAP-VER-006](versions-dependencies.md).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-DB-001 | SAP HANA Cloud is the production database | High | SAP-REC | Both |
| CAP-DB-002 | Development databases never serve production | High | SAP-REQ | Both |
| CAP-DB-003 | Use the current `@cap-js` database services — no direct driver dependencies | High | SAP-REC | Node.js |
| CAP-DB-004 | Access data through CQL/CQN; native SQL only as a documented exception | High | SAP-REC | Both |
| CAP-DB-005 | No UNIONs or JOINs in CQN — use path expressions and infix filters | Medium | SAP-REQ | Node.js |
| CAP-DB-006 | Request localized data explicitly with `SELECT.localized` | Medium | SAP-REQ | Node.js |
| CAP-DB-007 | Journal-based schema evolution for large-volume HANA entities | Medium | SAP-REC | Both |
| CAP-DB-008 | Use framework locking primitives for pessimistic concurrency | Medium | SAP-REC | Both |
| CAP-DB-009 | Do Decimal/Int64 arithmetic in the database | Medium | SAP-REC | Node.js |
| CAP-DB-010 | Prefer the static model style in CAP Java queries | Low | SAP-REC | Java |

---

## CAP-DB-001 — SAP HANA Cloud is the production database

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-001 |
| **Title** | SAP HANA Cloud is the production database |
| **Category** | Data access & persistence |
| **Severity** | High |
| **Authority** | SAP-REC ("recommended for productive use"), with documented validation constraints |
| **Applicability** | All projects with a production deployment target |
| **Runtime** | Both |
| **CAP version** | ⏱ Version-sensitive: CAP is validated against the **latest maintained QRC** of SAP HANA Cloud only; PostgreSQL lacks multitenancy/extensibility "yet" (re-verify at releases) |
| **Status** | Active |
| **Related rules** | CAP-DB-002, CAP-DB-007, CAP-MT-003 (MT requires HANA), CAP-ARCH-007 (deviation = ADR) |
| **Last verified** | 2026-08-11 |

### Rule statement
Production persistence MUST be SAP HANA Cloud — SAP's standard, "recommended for productive use" database with full schema-evolution and multitenancy support — unless a documented edge-case decision selects PostgreSQL (an ADR per CAP-ARCH-007 recording that multitenancy and extensibility "aren't yet supported on PostgreSQL"). Non-validated HANA variants (on-premise, HANA-as-a-Service) MUST NOT be assumed compatible: SAP validates only SAP HANA Cloud, latest maintained QRC.

### Rationale
SAP: "SAP HANA Cloud is supported as the CAP standard database and recommended for productive use with full support for schema evolution and multitenancy"; in production "SAP HANA is used by default", PostgreSQL "in edge cases"; "CAP isn't validated with variants other than SAP HANA Cloud" and only "against the latest maintained QRC version". **High:** running production on an unvalidated database or variant means undefined behavior in exactly the code paths CAP doesn't test — a production-reliability decision, though not itself an immediate integrity failure. PostgreSQL-in-production criteria remain ORG gap G-25.

### Evidence expected in code
`cds add hana` artifacts (Node: `@cap-js/hana` under `[production]`; Java: `cds-feature-hana`), HDI/HANA resources in `mta.yaml`/chart; or a PostgreSQL edge-case ADR.

### Detection guidance
1. Resolve the production database: Node — `cds.requires.db` under `[production]` in package.json/`.cdsrc.json`; Java — features in `pom.xml` plus binding types in `mta.yaml`/values.
2. HANA Cloud configured → PASS.
3. PostgreSQL configured → locate the ADR (edge-case rationale, single-tenant confirmation) → present → PASS with observation; absent → FAIL.
4. Any other production database (or a dev database — escalate to CAP-DB-002) → FAIL.
5. Multitenant project (per CAP-MT profile) on PostgreSQL → FAIL (documented unsupported combination).
6. Report configuration locations with file:line.

### Good example
```jsonc
// package.json — HANA in production, SQLite for the inner loop
{ "cds": { "requires": { "db": "sql",
    "[production]": { "db": { "kind": "hana" } } } } }
```

### Bad example
```text
Multitenant SaaS configured with @cap-js/postgres in production —
a combination SAP documents as not (yet) supported; no ADR anywhere.
```

### Exception guidance
PostgreSQL for documented edge cases (single-tenant, landscape constraints) with an ADR referencing G-25. No exception for unvalidated HANA variants without an explicit risk acceptance naming SAP's validation statement.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/hana (standard database; recommended for productive use; QRC validation scope; variants not validated)
- https://cap.cloud.sap/docs/guides/databases/ (HANA by default in production; PostgreSQL edge cases)
- https://cap.cloud.sap/docs/guides/databases/postgres (multitenancy/extensibility "aren't yet supported")

---

## CAP-DB-002 — Development databases never serve production

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-002 |
| **Title** | Development databases never serve production |
| **Category** | Data access & persistence |
| **Severity** | High |
| **Authority** | SAP-REQ ("not fit for production") |
| **Applicability** | All projects with a production deployment target |
| **Runtime** | Both (SQLite = Node.js dev default; H2 = Java dev default) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-DB-001, CAP-DB-008 (locking parity limits), CAP-CDS-009 (temporal parity), CAP-SRV-009 (streaming parity) |
| **Last verified** | 2026-08-11 |

### Rule statement
SQLite and H2 MUST NOT be the production database: SAP states SQLite "is not fit for production" (sole documented exception: in-memory caches, not the primary business database). Dev/prod parity is maintained through the database-agnostic services — but development code MUST NOT silently rely on behaviors that differ: pessimistic locking (unsupported on SQLite; H2 exclusive-only), media streaming (SQLite reads LargeBinary whole into memory), temporal time-travel (unsupported on SQLite). Where such features matter, hybrid tests against the production database are required.

### Rationale
SAP: "SQLite is mostly intended to speed up development, but is not fit for production", with the explicit in-memory-cache carve-out; the database services exist so HANA can be swapped with SQLite/H2 "during development" without model/implementation changes. In-memory dev databases lack the concurrency, durability, and feature surface of the production database — shipping them to production is a reliability and data-durability failure. **High:** direct production risk, but loud (deployment/config-level) rather than a silent corruption vector.

### Evidence expected in code
Production profile resolving to HANA (or documented PostgreSQL); SQLite/H2 confined to default/dev/test profiles; hybrid-test coverage for parity-sensitive features.

### Detection guidance
1. Resolve the effective production database (as in CAP-DB-001 step 1); SQLite/H2 in any production-reaching profile → FAIL.
2. Check `mta.yaml`/chart: absence of any database resource for a persistent app (would fall back to dev DB) → FAIL.
3. Inventory parity-sensitive features in use: `forUpdate`/`lock()` calls, media entities, temporal queries → for each, check hybrid/cloud test coverage exists (`cds bind --exec` tests, pipeline stage) → missing → report as missing-evidence element (not automatic FAIL of this rule).
4. In-memory-cache exception: SQLite used *alongside* the primary DB for caching → PASS with observation (documented carve-out).
5. Report with file:line.

### Good example
```jsonc
{ "cds": { "requires": {
    "db": "sql",                                  // sqlite in-memory: dev/test
    "[production]": { "db": { "kind": "hana" } }  // real DB in production
} } }
```

### Bad example
```jsonc
// production profile absent — deployed app falls back to SQLite,
// losing durability, concurrency, locking, and streaming
{ "cds": { "requires": { "db": { "kind": "sqlite", "credentials": { "url": "db.sqlite" } } } } }
```

### Exception guidance
Only the documented in-memory-cache scenario, recorded at the configuration. No exception for H2 in production (SAP documents it as the Java dev/test database).

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/sqlite ("not fit for production"; in-memory-cache exception)
- https://cap.cloud.sap/docs/guides/databases/ (dev swap of HANA with SQLite/H2)
- https://cap.cloud.sap/docs/guides/services/served-ootb (locking limitations on SQLite/H2)

---

## CAP-DB-003 — Use the current `@cap-js` database services — no direct driver dependencies

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-003 |
| **Title** | Use the current `@cap-js` database services — no direct driver dependencies |
| **Category** | Data access & persistence |
| **Severity** | High |
| **Authority** | SAP-REC (migration "highly recommended"; driver prohibition is normative wording: "should not") |
| **Applicability** | Node.js projects |
| **Runtime** | Node.js (Java: database features via Maven dependencies is the documented mechanism — no counterpart rule needed) |
| **CAP version** | ⏱ `@cap-js/{sqlite,postgres,hana}` GA since cds 8, default since; major lines must pair with `@sap/cds` (e.g., `@cap-js/*` 3.x with cds 10 — see [version register](../../docs/version-management.md)) |
| **Status** | Active |
| **Related rules** | CAP-DB-005, CAP-DB-006 (behavioral consequences of the new services), CAP-DB-002 |
| **Last verified** | 2026-08-11 |

### Rule statement
Node.js projects MUST use the current `@cap-js` database services (`@cap-js/sqlite`, `@cap-js/hana`, `@cap-js/postgres`) — auto-configured via the `cds-plugin` mechanism — and MUST NOT add direct dependencies on driver packages (SAP: "You don't need to — and should not — add direct dependencies to driver packages, like `hdb` or `sqlite3` anymore") or hand-maintain `cds.requires.db` beyond deliberate overrides. Projects still on the old built-in database services MUST have a migration plan (SAP: migration "highly recommended" since cds 8).

### Rationale
The `@cap-js` services are the maintained line, auto-configured, and behaviorally distinct (CAP-DB-005/-006); direct driver dependencies duplicate what the service packages manage and drift out of the supported version pairing. On current majors, mixing old services or stray drivers with cds 10 violates the documented major-line pairing. **High:** an unmaintained persistence stack in production accrues unfixed defects; version-pairing violations are unsupported configurations. (Authority SAP-REC on migration — SAP documents no removal date — with the driver clause carrying normative "should not" wording.)

### Evidence expected in code
`@cap-js/*` database packages in `package.json` matching the profile matrix; no `hdb`/`sqlite3`/`better-sqlite3` direct dependencies (unless a documented cds-10 opt-in, e.g., retaining `better-sqlite3` explicitly); minimal `cds.requires.db` config.

### Detection guidance
1. Read `package.json` dependencies: presence of `@cap-js/sqlite` (dev) and `@cap-js/hana` (production) or the project's documented equivalents.
2. Direct `hdb`/`sqlite3` dependencies alongside the service packages → FAIL (name the package to remove).
3. Old built-in service usage (legacy `cds.requires.db.kind` configs without `@cap-js` packages on cds ≥ 8) → FAIL unless a dated migration plan exists.
4. Check major-line pairing: `@cap-js/*` major vs `@sap/cds` major per the version register → mismatch → FAIL (cross-report to CAP-VER-004).
5. Report with file:line into `package.json`.

### Good example
```jsonc
{ "dependencies": { "@sap/cds": "^10", "@cap-js/hana": "^3" },
  "devDependencies": { "@cap-js/sqlite": "^3" } }
```

### Bad example
```jsonc
// direct drivers next to (or instead of) the service packages
{ "dependencies": { "@sap/cds": "^10", "hdb": "^0.19", "sqlite3": "^5" } }
```

### Exception guidance
Documented opt-ins per release notes (e.g., explicitly retaining `better-sqlite3` under cds 10 instead of `node:sqlite`) are compliant when recorded. Legacy projects mid-migration operate under a dated exception.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/new-dbs ("should not add direct dependencies to driver packages"; migration "highly recommended"; GA with cds 8)
- https://cap.cloud.sap/docs/guides/databases/sqlite (cds-plugin auto-configuration)

---

## CAP-DB-004 — Access data through CQL/CQN; native SQL only as a documented exception

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-004 |
| **Title** | Access data through CQL/CQN; native SQL only as a documented exception |
| **Category** | Data access & persistence |
| **Severity** | High |
| **Authority** | SAP-REC (CQL/CQN is the documented data-access model; Java's documented native-SQL escape hatch is Spring `JdbcTemplate`) |
| **Applicability** | All custom data access |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | **CAP-SEC-013** (injection rules apply to *any* query construction — not restated here), CAP-ARCH-005, CAP-DB-005, CAP-DB-010 |
| **Last verified** | 2026-08-11 |

### Rule statement
Custom code MUST access data through CAP's query APIs — `cds.ql`/CQL (Node.js), the CQN `Select/Insert/Upsert/Update/Delete` builders (Java) — executed via CAP services. Native SQL is permitted only where CDS/CQN cannot express the operation (e.g., HANA stored procedures, non-CDS artifacts), using the documented mechanism (Java: Spring `JdbcTemplate`), isolated in one place, recorded per CAP-ARCH-007, and fully subject to CAP-SEC-013's injection rules. Criteria for when native SQL is justified remain ORG gap G-24.

### Rationale
CQL/CQN is what makes CAP's guarantees work: database-agnostic execution (CAP-DB-002 parity), automatic tenant-aware data access, localized handling, push-down optimization, and prepared-statement parameterization. Native SQL bypasses all of it — every native statement is vendor-locked, tenant-handling is manual, and the injection surface widens (CAP-SEC-013). **High:** systematic bypassing of the data-access abstraction erodes integrity and portability guarantees across the codebase; isolated justified usage is the managed exception.

### Evidence expected in code
Handlers using `SELECT/INSERT/UPDATE/DELETE` builders / `srv.run`; native SQL confined to documented, isolated modules with ADR reference.

### Detection guidance
1. Search `srv/**` for native SQL: raw SQL strings passed to run/execute APIs, Java `JdbcTemplate`/`DataSource` usage, Node direct driver calls.
2. For each hit: locate the justification (ADR per CAP-ARCH-007, inability of CQN to express the operation) → absent → FAIL with file:line.
3. Verify justified native SQL is isolated (single module/class, not scattered) and parameterized per CAP-SEC-013 (cross-report injection findings there).
4. Verify justified native SQL handles tenancy explicitly in multitenant projects (cross-ref CAP-MT-003) → unaddressed → FAIL.
5. Report each site with the CQN alternative where one exists.

### Good example
```java
// CQN builder through the persistence service — portable, tenant-aware
Result r = db.run(Select.from(BOOKS).columns(b -> b.title()).where(b -> b.stock().gt(0)));
```

### Bad example
```js
// raw vendor SQL in a handler — bypasses tenancy, localization,
// portability; and (here) violates CAP-SEC-013 too
const rows = await db.run(`SELECT * FROM MY_BOOKS WHERE TITLE LIKE '%${req.data.q}%'`);
```

### Exception guidance
Documented cases CQN cannot express (stored procedures, DB-specific admin operations, non-CDS tables) via the runtime's documented mechanism, isolated and recorded (G-24 governs the org criteria). No exception for convenience SQL where CQN suffices.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-ql (Node.js query API)
- https://cap.cloud.sap/docs/java/working-with-cql/query-api (Java query builders)
- https://cap.cloud.sap/docs/java/working-with-cql/query-execution (no dedicated native-SQL API — Spring `JdbcTemplate` where needed)

---

## CAP-DB-005 — No UNIONs or JOINs in CQN — use path expressions and infix filters

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-005 |
| **Title** | No UNIONs or JOINs in CQN — use path expressions and infix filters |
| **Category** | Data access & persistence |
| **Severity** | Medium |
| **Authority** | SAP-REQ (documented dropped capability: "we dropped support of UNIONs and JOINs in CQN queries") |
| **Applicability** | Node.js projects on the `@cap-js` database services |
| **Runtime** | Node.js |
| **CAP version** | ⏱ Behavior of the `@cap-js` database services (default since cds 8; verified on current docs) |
| **Status** | Active |
| **Related rules** | CAP-DB-003, CAP-CDS-005 (associations enable the path expressions), CAP-SEC-013 |
| **Last verified** | 2026-08-11 |

### Rule statement
CQN queries MUST NOT use UNIONs or JOINs — the current database services do not support them ("we dropped support of UNIONs and JOINs in CQN queries"). Cross-entity reads are expressed with path expressions and infix filters over modeled associations (SAP: "Use path expressions instead of joins"); set-union requirements are remodeled (single entity with discriminator — cross-ref CAP-CDS guidance) or, as a last resort, become a documented native-SQL exception per CAP-DB-004. Note the related documented behavior: the services ignore virtual elements in result sets.

### Rationale
This is a hard capability boundary of the supported persistence stack, most relevant to code migrated from the old database services: queries that once worked now fail. Path expressions also preserve push-down and model semantics that hand-built joins bypass. **Medium:** violations fail loudly at runtime (defects, not silent corruption); the migration-era codebase scan is the review value.

### Evidence expected in code
Handler queries using association paths (`SELECT.from('Orders').where('items.book.genre =', …)`, `.columns(o => o.items(i => i.book('title')))`) rather than JOIN/UNION CQN constructs.

### Detection guidance
1. Search `srv/**/*.js|ts` for CQN JOIN/UNION construction: `.join(`, `UNION`, `SELECT.union`, hand-built CQN objects with `join`/`SET` operators.
2. Each hit targeting the primary database service → FAIL with file:line and the path-expression alternative (or remodeling recommendation).
3. Check code reading virtual elements from result sets (they're ignored by the services) → flag as ineffective read → FAIL.
4. Legacy migration check: projects recently migrated from old DB services → prioritize this scan.
5. NOT APPLICABLE for Java projects and for documented native-SQL modules (governed by CAP-DB-004).

### Good example
```js
// path expression + infix filter instead of a JOIN
const titles = await SELECT.from('Orders')
  .columns(o => { o.ID, o.items(i => i.book(b => b.title)) })
  .where({ 'items.book.genre_ID': genreID });
```

### Bad example
```js
// hand-built JOIN CQN — unsupported by @cap-js database services
const q = SELECT.from('Orders as o').join('Books as b').on('o.book_ID = b.ID');
```

### Exception guidance
Genuine set operations or join shapes inexpressible via paths go through CAP-DB-004's native-SQL exception (documented, isolated) — never through unsupported CQN.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/new-dbs (dropped UNION/JOIN support; path expressions; virtual elements ignored)

---

## CAP-DB-006 — Request localized data explicitly with `SELECT.localized`

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-006 |
| **Title** | Request localized data explicitly with `SELECT.localized` |
| **Category** | Data access & persistence |
| **Severity** | Medium |
| **Authority** | SAP-REQ (documented behavior: plain `SELECT.from` returns non-localized data) |
| **Applicability** | Node.js custom code reading `localized` elements outside generic handlers |
| **Runtime** | Node.js (generic READ handlers localize automatically on both runtimes) |
| **CAP version** | ⏱ Behavior of the `@cap-js` database services (since cds 8) |
| **Status** | Active |
| **Related rules** | CAP-CDS-008 (`localized` modeling), CAP-DB-003 |
| **Last verified** | 2026-08-11 |

### Rule statement
Custom Node.js code that needs locale-dependent texts MUST read them via `SELECT.localized(...)` — with the current database services, plain `SELECT.from(...)` returns **non-localized** data. Code paths serving end users (custom handlers assembling responses, exports, notifications) make this choice deliberately per read.

### Rationale
SAP documents the split explicitly (`SELECT.from(Books) //> non-localized data` vs `SELECT.localized(Books) //> localized data`). The failure mode is silent: queries succeed and return default-language texts, surfacing as wrong-language UI content for non-default locales — typically missed in single-locale testing. **Medium:** functional correctness defect, scoped to localized reads in custom code.

### Evidence expected in code
`SELECT.localized` in custom read paths that feed user-visible localized elements; plain `SELECT.from` on localized entities only where default-language/technical reads are intended.

### Detection guidance
1. From the model, list entities with `localized` elements (per CAP-CDS-008 detection).
2. Search `srv/**/*.js|ts` for `SELECT.from`/`SELECT.one` on those entities selecting localized elements.
3. For each, determine the consumer: user-facing output (handler results, documents, messages) with plain `SELECT.from` → FAIL with file:line; technical/internal reads (matching on codes, admin exports of originals) → PASS with note.
4. Verify at least one test exercises a non-default locale on custom localized read paths (missing → evidence gap note).
5. NOT APPLICABLE if no localized elements exist or no custom reads touch them.

### Good example
```js
srv.on('READ', 'FeaturedBooks', async req =>
  SELECT.localized(Books).where({ featured: true }));   // user-facing: localized
```

### Bad example
```js
// user-facing catalog assembled with plain SELECT — always default language
srv.on('READ', 'FeaturedBooks', async () =>
  SELECT.from(Books).where({ featured: true }));
```

### Exception guidance
Deliberate default-language reads (comparisons, replication, admin views of originals) are compliant — a short comment at the site prevents re-flagging.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/new-dbs (`SELECT.from` non-localized vs `SELECT.localized`)

---

## CAP-DB-007 — Journal-based schema evolution for large-volume HANA entities

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-007 |
| **Title** | Journal-based schema evolution for large-volume HANA entities |
| **Category** | Data access & persistence |
| **Severity** | Medium |
| **Authority** | SAP-REC (selective annotation guidance; "must be checked into the version control system" is documented REQ wording for the generated files) |
| **Applicability** | HANA-deployed projects with large-data-volume entities; NOT APPLICABLE otherwise |
| **Runtime** | Both |
| **CAP version** | All currently supported versions (`.hdbmigrationtable` mechanism) |
| **Status** | Active |
| **Related rules** | CAP-DB-001, CAP-MT-005 (tenant upgrades run schema evolution) |
| **Last verified** | 2026-08-11 |

### Rule statement
Entities expected to hold large data volumes on SAP HANA SHOULD be annotated `@cds.persistence.journal`, switching them to `.hdbmigrationtable`-based schema evolution (SAP: "Schema updates using .hdbtable deployments are a challenge for tables with large data volume"). The generated `.hdbmigrationtable` files are source files and MUST be checked into version control (documented requirement).

### Rationale
Default `.hdbtable` deployment recreates/copies tables on incompatible changes — for big tables that means long deployments and outage risk exactly where it hurts (and multiplied per tenant in MT upgrades, CAP-MT-005). The journal mechanism applies incremental migrations instead. **Medium:** an operational/deployment-robustness safeguard; absence bites at scale, not immediately.

### Evidence expected in code
`@cds.persistence.journal` on high-volume entities; generated `db/src/**/*.hdbmigrationtable` files committed to the repository.

### Detection guidance
1. Identify high-volume entities (requirements/NFRs, entity nature: transactions, logs, line items).
2. Check annotations: high-volume entities without `@cds.persistence.journal` → FAIL/observation depending on volume evidence strength.
3. If the annotation is used: verify `.hdbmigrationtable` files exist under version control (not gitignored) → gitignored/missing → FAIL (documented "must").
4. NOT APPLICABLE for non-HANA targets or uniformly small data.
5. Report per entity.

### Good example
```cds
@cds.persistence.journal        // millions of rows — incremental migrations
entity OrderItems : cuid { /* … */ }
```
```text
db/src/gen/OrderItems.hdbmigrationtable   ← committed source file
```

### Bad example
```text
A 200M-row transaction table on default .hdbtable deployment —
every incompatible model change triggers a full-table copy during
the (per-tenant!) upgrade window.
```

### Exception guidance
Deliberately staying on `.hdbtable` for a large entity (e.g., planned rebuild windows) is acceptable when recorded with the operational plan. Small/medium tables need no annotation — that is the default's design.

### SAP reference
- https://cap.cloud.sap/docs/guides/databases/hana (`@cds.persistence.journal`; large-volume challenge; migration files "must be checked into the version control system")

---

## CAP-DB-008 — Use framework locking primitives for pessimistic concurrency

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-008 |
| **Title** | Use framework locking primitives for pessimistic concurrency |
| **Category** | Data access & persistence |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented mechanism with documented limitations) |
| **Applicability** | Code requiring pessimistic locking; NOT APPLICABLE where optimistic control (CAP-SRV-008) or drafts suffice |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SRV-008 (optimistic ETags — the default choice), CAP-DB-002 (dev-parity limits), CAP-TXN-001 (locks live until transaction end) |
| **Last verified** | 2026-08-11 |

### Rule statement
Where pessimistic locking is genuinely required, code MUST use the framework primitives — `SELECT … forUpdate()` (Node.js) / `Select.…lock()` with mode, timeout, and wait strategy (Java, incl. `SKIP_LOCKED` for queue-like processing) — whose locks hold "until the end of the transaction by commit or rollback". Hand-rolled locking (lock flags/columns, lock tables, application semaphores) MUST NOT be used. Documented limitations apply: locking works on domain entities (table rows), "not possible for projections and views"; "not supported by SQLite. H2 supports exclusive locks only" — production-database tests required for lock-dependent logic.

### Rationale
The framework primitives delegate to real database row locks with correct transactional scope; hand-rolled lock flags are not crash-safe (orphaned locks after failures), not enforced by the database, and invisible to other access paths. The documented dev-parity limitation means SQLite tests cannot validate locking logic at all. **Medium:** correctness of concurrent flows where used; the common case is served by optimistic control (CAP-SRV-008), making this a scoped rule.

### Evidence expected in code
`forUpdate()`/`.lock()` usage on entities (not projections) where pessimistic semantics are needed; no lock-flag columns/tables; hybrid tests for lock behavior.

### Detection guidance
1. Search for hand-rolled locking: model elements like `lockedBy`/`lockedAt`/`isLocked` with handler enforcement, dedicated lock entities, mutex libraries around data access → each → FAIL (name the framework primitive).
2. Search `forUpdate(`/`.lock(` usages: verify targets are domain entities, not projections/views (documented limitation) → violation → FAIL.
3. Verify lock usage sits inside a transaction whose scope matches the protected operation (locks release at commit/rollback — cross-check CAP-TXN-001).
4. Check test strategy: lock-dependent logic tested only on SQLite → flag missing hybrid/production-DB test (evidence gap).
5. NOT APPLICABLE if no pessimistic locking need exists (state which mechanism covers concurrency instead).

### Good example
```java
// queue-like processing: competing workers skip locked rows
Result r = db.run(Select.from(TASKS).where(t -> t.status().eq("open"))
    .lock(LockMode.EXCLUSIVE, WaitStrategy.SKIP_LOCKED));
```

### Bad example
```cds
entity Tasks : cuid {
  lockedBy : String;    // hand-rolled lock flag: not crash-safe,
  lockedAt : Timestamp; // not DB-enforced, orphaned after failures
}
```

### Exception guidance
Cross-request "locks" (a user editing for minutes) are not database locks — use drafts (CAP-SRV-007, which lock during edit sessions) or optimistic ETags (CAP-SRV-008); that is redirection, not an exception. Genuine distributed locks across systems are integration architecture (ADR per CAP-ARCH-007).

### SAP reference
- https://cap.cloud.sap/docs/guides/services/served-ootb (pessimistic locking; transaction-scoped; limitations: no projections/views, no SQLite, H2 exclusive-only)
- https://cap.cloud.sap/docs/java/working-with-cql/query-api (`lock()` modes, timeout, `SKIP_LOCKED`)

---

## CAP-DB-009 — Do Decimal/Int64 arithmetic in the database

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-009 |
| **Title** | Do Decimal/Int64 arithmetic in the database |
| **Category** | Data access & persistence |
| **Severity** | Medium |
| **Authority** | SAP-REC (best-practice guidance grounded in the documented string-return behavior) |
| **Applicability** | Node.js code computing on `Decimal`/`Int64` elements |
| **Runtime** | Node.js (Java's typed accessors use `BigDecimal`/`long` — no counterpart rule needed) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-DB-004, CAP-SEC-013 (the `set('…')` expression form must not carry user input) |
| **Last verified** | 2026-08-11 |

### Rule statement
Calculations on `Decimal`/`Int64` elements MUST be pushed to the database (e.g., ``UPDATE(Books).set('stock = stock + 1')`` or `set` with `+=`-style expressions), not performed in JavaScript: CAP returns these values as **strings** because "JavaScript's Number type cannot represent these values without risk of losing precision" — SAP: "Doing such calculations in JavaScript risks silently losing precision — do them in the database instead." Where in-process computation is unavoidable, a decimal library (not `Number`) is used and the choice recorded.

### Rationale
The string return is a documented, deliberate protection; casting to `Number` for arithmetic silently defeats it — monetary totals and large IDs corrupt without errors. Database-side expressions are also atomic (`stock = stock + 1` avoids read-modify-write races, cross-ref CAP-TXN-001). **Medium:** a silent-corruption class defect but scoped to code doing in-process arithmetic on such fields; escalate findings on monetary elements in the report.

### Evidence expected in code
DB-side expression updates for numeric mutations; no `Number(...)`/`parseFloat(...)` arithmetic on Decimal/Int64 elements written back to the database.

### Detection guidance
1. From the model, list `Decimal`/`Int64` elements.
2. Search `srv/**/*.js|ts` for reads of those elements followed by arithmetic (`Number(`, `parseFloat(`, `+`/`*` on the values) whose results are persisted or returned → FAIL with file:line (escalate severity note if the element is monetary).
3. Verify mutation patterns use DB-side expressions (`.set('elem = elem + …')` with **literal/validated** expressions only — user input in the expression string → cross-report CAP-SEC-013).
4. Where in-process decimal math exists: check for a decimal library and a recorded decision → present → PASS with note.
5. NOT APPLICABLE if the model has no Decimal/Int64 elements or no custom code computes on them.

### Good example
```js
// atomic, precision-safe: computed in the database
await UPDATE(Books, ID).set('stock = stock - 1');
```

### Bad example
```js
const { price } = await SELECT.one.from(Books, ID, b => b.price); // "9999999.99" (string)
const total = Number(price) * quantity;                            // precision at risk
await UPDATE(Orders, oID).with({ total });                         // silently corrupted
```

### Exception guidance
Unavoidable in-process computation (complex pricing engines) with a proper decimal library and a recorded decision. Display-only formatting of the string values is unaffected.

### SAP reference
- https://cap.cloud.sap/docs/node.js/best-practices (string return; "risks silently losing precision — do them in the database instead"; `set('stock = stock + 1')` example)

---

## CAP-DB-010 — Prefer the static model style in CAP Java queries

| Field | Value |
|---|---|
| **Rule ID** | CAP-DB-010 |
| **Title** | Prefer the static model style in CAP Java queries |
| **Category** | Data access & persistence |
| **Severity** | Low |
| **Authority** | SAP-REC ("it's recommended to use the static style when implementing business logic") |
| **Applicability** | CAP Java business-logic code building queries |
| **Runtime** | Java |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-DB-004, CAP-SEC-013 (static style also reduces string-built query surface) |
| **Last verified** | 2026-08-11 |

### Rule statement
CAP Java business logic SHOULD build queries in the static model style (generated model classes: `Select.from(BOOKS).columns(b -> b.title())`), reserving the dynamic string-based style for genuinely generic code — per SAP's recommendation, for design-time name checking, IDE completion, and type-safe predicate composition.

### Rationale
SAP: "it's recommended to use the static style when implementing business logic that requires accessing particular elements of entities", with the dynamic style positioned for generic code. Static style catches model drift at compile time (renamed elements break the build, not production) and structurally discourages string assembly (CAP-SEC-013 adjacency). **Low:** code-quality/maintainability; both styles are supported and correct.

### Evidence expected in code
Generated model classes (cds-maven-plugin codegen) imported and used in handlers; dynamic string style confined to generic utilities.

### Detection guidance
1. Verify code generation is configured (cds-maven-plugin generating the static model).
2. Sample business-logic handlers: entity/element access via string literals (`Select.from("my.bookshop.Books").columns("title")`) where static classes exist → observation; systematic across the codebase → FAIL (Low).
3. Generic/framework-style utilities using dynamic style → compliant by SAP's positioning; note as such.
4. Report representative sites.

### Good example
```java
Result r = db.run(Select.from(BOOKS)
    .columns(b -> b.title(), b -> b.stock())
    .where(b -> b.stock().gt(0)));
```

### Bad example
```java
// stringly-typed in plain business logic — no design-time checking
Result r = db.run(Select.from("my.bookshop.Books")
    .columns("title", "stok")        // typo ships to production
    .where(CQL.get("stock").gt(0)));
```

### Exception guidance
Generic components operating over arbitrary entities (auditing, replication utilities) legitimately use the dynamic style — SAP's own positioning; note the purpose at the class.

### SAP reference
- https://cap.cloud.sap/docs/java/working-with-cql/query-api (static style recommended for business logic; dynamic for generic code)
