# Candidate Rule Inventory

Researched candidate rules for the Phase 2 Layer 2 catalog, derived from the [SAP source map](sap-cap-sources.md) (all sources verified 2026-08-11). **These are candidates, not normative rules** — they become citable only when promoted into [standards/rules/](../standards/rules/README.md) with all [template](../templates/rule-template.md) fields completed.

Severity and IDs are provisional. Authority: REQ = SAP-documented requirement, REC = SAP-documented recommendation, ORG = needs org decision (see [research-gaps.md](research-gaps.md)). Runtime: N/J/B. ⏱ = version-sensitive (see [version-management](../docs/version-management.md)).

**Total: 101 candidates** across 20 categories.

> **Disposition status:** the **CAP-SEC**, **CAP-MT** (Batch 1), **CAP-ARCH**, **CAP-CDS**, **CAP-SRV** (Batch 2), the **CAP-DB**, **CAP-TXN**, **CAP-EVT** (Batch 3), the **CAP-LOGIC**, **CAP-INT** (Batch 4), the **CAP-TEST**, **CAP-ERR**, **CAP-LOG** (Batch 5), **CAP-PERF**, **CAP-EXT**, **CAP-PRIV** (Batch 6, incl. deferred CAP-SEC #17), and **CAP-DEP**, **CAP-CICD**, **CAP-VER**, **CAP-OPS** (Batch 7, final) sections below — plus cross-category items CAP-OPS #3, CAP-LOGIC #1/#5, CAP-PERF #6 — have been formally dispositioned: see [candidate-dispositions.md](candidate-dispositions.md) and the normative rules in the [rule catalog](../standards/rules/README.md). Dispositioned entries below are retained as research history; the catalog files are authoritative.

## CAP-ARCH — Architecture & project structure (8)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Standard project layout (db/srv/app), convention over configuration | High | REC | B | Use the `cds init` layout and defaults rather than overriding them | get-started/ |
| 2 | No abstraction layers on top of CAP | Critical | REC | B | Never wrap CAP APIs in own abstractions, Active Records, repository layers, or DIY hexagonal scaffolding | about/bad-practices ⚠404 — re-verify URL |
| 3 | Single-purpose services per use case | Critical | REC | B | One service per use case/user group; never one service exposing the domain 1:1 | guides/services/providing-services |
| 4 | Services expose tailored projections | High | REC | B | Service entities are denormalized projections excluding internal/sensitive fields | guides/services/providing-services |
| 5 | Stateless services, passive data; no element-level frameworks | High | REC | B | No behavior-attached data objects; no element-level determination/validation frameworks | get-started/concepts |
| 6 | Platform/protocol-agnostic application code | High | REC | B | Consume DB/messaging/auth/remote services only via CAP's uniform service APIs | get-started/concepts |
| 7 | Start modulithic; cut microservices late, by configuration | Medium | REC | B | Defer deployment-unit cuts; same code deploys as monolith or microservices | get-started/features, guides/deploy/microservices |
| 8 | CSV seed data convention (`db/data/<ns>-<Entity>.csv`) | Medium | REC | B | Follow the naming convention required for zero-config data loading | get-started/bookshop |

## CAP-CDS — Domain modeling & CDS (13)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Entity/element naming casing (Capitalized entities, lowercase elements) | Medium | REC | B | Follow SAP naming conventions | guides/domain/ |
| 2 | Plural entities, singular types | Medium | REC | B | `Authors` entity, `Genre` type | guides/domain/ |
| 3 | Concise names, no context repetition, `ID` for technical keys | Low | REC | B | `Authors.name`, not `authorName` | guides/domain/ |
| 4 | Canonical UUID keys via `cuid` | High | REC | B | Prefer `cuid` aspect; DB sequences only for genuinely high volumes | guides/domain/ |
| 5 | UUIDs are opaque | High | REQ | B | Never validate/normalize/convert UUID values | guides/domain/ |
| 6 | Use `@sap/cds/common` aspects & types | High | REC | B | `managed`/`cuid`/`temporal`/`Country`/`Currency`/`Language` over hand-rolled equivalents | cds/common |
| 7 | Prefer managed associations | High | REC | B | Managed to-one; `on <assoc>.<backlink> = $self` for to-many | guides/domain/ |
| 8 | Compositions for contained-in; no composition-of-one for entities | High | REC | B | Managed compositions of aspects preferred; composition-of-one discouraged | cds/cdl |
| 9 | Separate concerns via aspects in dedicated files | High | REC | B | Authorization/UI/localization annotations in separate files | cds/aspects, guides/domain/ |
| 10 | `localized` keyword for translatable text | High | REC | B | Never hand-model translation tables; BCP 47 underscores; keys not associations | guides/uis/localized-data |
| 11 | Temporal data via `temporal` aspect; writes in custom handlers only | Medium | REQ (writes) | B | No generic write support for time slices | guides/domain/temporal-data |
| 12 | Flat models; custom types only with reuse ratio | Medium | REC | B | Avoid nested structured types | guides/domain/ |
| 13 | Stable namespaces (no org/project codenames) | Medium | REC | B | Reverse-domain style; avoid short-lived tokens | guides/domain/ |

## CAP-SRV — Services & APIs (10)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Prefer generic providers over reimplementation | High | REC | B | Use OOTB CRUD/deep-write/search/pagination/sorting | guides/services/served-ootb |
| 2 | Declarative input validation first (`@mandatory`, `@assert.*`) | High | REC | B | Annotations before imperative validation ⏱ (open intervals cds 8.5+/Java 3.5+) | guides/services/constraints |
| 3 | Declarative restrictions (`@readonly`/`@insertonly`); never `@readonly` on keys | Medium | REC/REQ | B | Restrict declaratively, not in handlers | constraints, providing-services |
| 4 | Actions modify, functions read | Medium | REQ | B | Never modify data in a function | guides/services/custom-actions |
| 5 | OData V4 default; V2 only for legacy UIs | High | REQ (V2 deprecated) | B | ⏱ Node V2 via community adapter; Java built-in | guides/protocols/odata |
| 6 | Explicit protocol exposure; `@protocol:'none'` for internal services | Medium | REC | B | Deliberate exposure decisions; relative `@path` | node.js/cds-serve, java/…/application-services |
| 7 | Framework draft handling via `@odata.draft.enabled` | High | REC | B | No custom draft persistence | guides/uis/fiori |
| 8 | Validate active entities, not only drafts | High | REQ | B | "Validations on draft entities alone are not sufficient" | guides/uis/fiori |
| 9 | Optimistic concurrency via `@odata.etag`; Java checks `rowCount()==0` | Medium | REC | B | Conflict detection on concurrently edited entities | served-ootb, java query-execution |
| 10 | Media via `@Core.MediaType` + streaming | High | REC (strong) | B | Never base64 buffers in memory; Java: close write streams | guides/services/media-data |

## CAP-LOGIC — Business logic & event handlers (8)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Declarative before imperative | Medium | REC | B | Check declarative options before writing handlers | guides/services/custom-code |
| 2 | Correct phase usage (before=validate, on=core, after=enrich) | High | REC | B | Node before/after run in parallel — no order assumptions | custom-code, node.js/core-services |
| 3 | `return super.init()` in Node service subclasses | High | REQ | N | Generic handlers register only then | node.js/core-services |
| 4 | Collect validation errors via `req.error`; abort via `req.reject` | Medium | REC | N | All input errors reported together | node.js/core-services |
| 5 | Always `await srv.emit()` and all async ops | High | REQ | N | Unawaited promises risk invalid tx state/deadlock | node.js/core-services, cds-tx |
| 6 | Java handler registration pattern + typed event contexts | High | REQ/REC | J | `@Component` + `EventHandler` + `@ServiceName`; typed contexts | java/event-handlers/ |
| 7 | Actions/functions need `@On` handlers in Java | High | REQ | J | No default On handlers exist | java/…/application-services |
| 8 | Domain logic on ApplicationService, technical on PersistenceService | Medium | REC | J | Correct service layering | java/…/application-services |

## CAP-DB — Data access & persistence (10)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | SAP HANA Cloud for production | High | REC | B | Standard production DB; PostgreSQL edge cases only (no MT/extensibility) ⏱ | guides/databases/hana, postgres |
| 2 | No SQLite/H2 in production | High | REQ | B | Dev/test only; keep dev/prod parity via agnostic services | guides/databases/sqlite, h2 |
| 3 | New `@cap-js/*` database services | High | REQ | N | ⏱ Old services deprecated since cds 8 | guides/databases/new-dbs |
| 4 | No direct driver dependencies (`hdb`, `sqlite3`) | Medium | REC | N | cds-plugin auto-configuration | guides/databases/sqlite |
| 5 | CQL/cds.ql over raw SQL; Java native SQL only via JdbcTemplate where CDS can't | High | REC | B | Preserve push-down, tenant isolation, portability | node.js/cds-ql, java query APIs |
| 6 | Injection-safe queries: no string concat, no parenthesized templates, `CQL.param()`, no input in hints | Critical | REQ | B | User input only as parameters; positive-list for structure | cds-ql, java query APIs, security/data-protection |
| 7 | Path expressions/infix filters, not CQN UNION/JOIN | High | REQ | N | ⏱ Dropped by new DB services | guides/databases/new-dbs |
| 8 | `SELECT.localized` for localized reads | Medium | REQ | N | ⏱ Plain SELECT returns non-localized since new DB services | guides/databases/new-dbs |
| 9 | `@cds.persistence.journal` for large HANA entities | Medium | REC | B | `.hdbmigrationtable` schema evolution | guides/databases/hana |
| 10 | Static model style in Java queries | Medium | REC | J | Compile-time checking over string style | java/working-with-cql/query-api |

## CAP-TXN — Transactions (5)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Rely on managed transactions in handlers | High | REC | B | No manual begin/commit/rollback in handlers | node.js/cds-tx, java changeset-contexts |
| 2 | Explicit `cds.tx()` only outside handlers; prefer functional form | Medium | REC | N | Background jobs/scripts only | node.js/cds-tx |
| 3 | No deprecated `cds.tx(req)` | Medium | REQ | N | ⏱ Context propagates via AsyncLocalStorage | node.js/cds-tx |
| 4 | ChangeSet API or Spring `@Transactional` in Java | Medium | REC | J | Both fully integrated; `beforeClose` to veto | java changeset-contexts |
| 5 | No distributed-atomicity assumptions | High | REQ (documented limitation) | B | Nested tx / ChangeSet members not committed atomically together | cds-tx, changeset-contexts |

## CAP-INT — Integration & remote services (7)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Import remote APIs via `cds import`; never copy CDS files | High | REQ | B | EDMX into `srv/external` | guides/services/consuming-services |
| 2 | OData V4 between CAP services | Medium | REC | B | EDMX as exchange format | consuming-services |
| 3 | Consumption views over direct remote-entity exposure | High | REC | B | Map remote definitions to own domain | guides/integration/calesi |
| 4 | No credentials in destination configuration | Critical | REQ | B | Programmatic/env provisioning; BTP Destination service | consuming-services |
| 5 | Mock remote services in development | Medium | REC | B | `cds mock`, CSV data; Java `--with-mocks` | consuming-services |
| 6 | Restrict cross local/remote expands | High | REC | B | Explicit handlers; reject on large result sets | consuming-services |
| 7 | Deliberate MT destination retrieval strategy | Medium | REC | B | Subscriber vs `alwaysProvider` decided consciously | consuming-services |

## CAP-EVT — Events & messaging (7)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Declare/emit events protocol-agnostically | High | REC | B | CDS events + `srv.emit`/`srv.on`; no broker code in app logic | guides/events/core-concepts |
| 2 | Keep persistent outbox enabled in production | Critical | REQ | B | Messages commit with business data | guides/events/event-queues |
| 3 | Idempotent event/queued handlers | Critical | REQ | B | Exactly-once NOT guaranteed | event-queues |
| 4 | No sensitive data in message headers; claims in payload | Critical | REQ | B | Queued processing runs privileged | event-queues |
| 5 | Dead-letter operations process | High | REC | B | Monitor, revive/delete exhausted messages (ORG thresholds) | event-queues |
| 6 | Event Hub as default broker for new BTP apps | Medium | REC | B | ⏱ "New default offering"; Event Mesh legacy | guides/events/event-hub |
| 7 | Local/file-based messaging for local dev | Medium | REC | B | Event Hub can't webhook to localhost | event-hub, event-mesh |

## CAP-MT — Multitenancy (5)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Streamlined MTX (`@sap/cds-mtxs`) only | Critical | REQ | B | ⏱ Old MTX maintenance-only (≤ Java 2.x) | guides/multitenancy/, old-mtx-migration |
| 2 | MTX sidecar: mandatory (Java), recommended for prod (Node) | High | REQ (J) / REC (N) | B | Isolate subscribe/upgrade load | multitenancy/mtxs, java/multitenancy |
| 3 | Strict per-tenant DB isolation; no bypassing tenant-aware access | Critical | REQ | B | HDI container per tenant; no static mutable state (J) / closure state (N) | guides/multitenancy/, security/data-protection |
| 4 | Idempotent subscribe handlers; technical roles never in business roles | Critical | REQ | B | `cds.Subscriber`/`mtcallback`/`emcallback` | multitenancy/mtxs |
| 5 | Upgrade all tenants before serving a new model version | Critical | REQ | B | Java `Deploy` main before app start; `cds-mtx upgrade '*'` | guides/multitenancy/, java/multitenancy |

## CAP-EXT — Extensibility (3)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Extension allowlist as deliberate artifact | High | REQ (default-forbidden) | B | Namespaces, `x_` prefixes, field caps (ORG content) | mtxs, extensibility/customization |
| 2 | Extension workflow: pull → local test → test tenant → prod | Medium | REC | B | `cds.ExtensionDeveloper` role; `cds pull`/`push` | extensibility/customization |
| 3 | Feature toggles: isolated features, uniform schema | High | REQ | B | No inter-feature `using`; all features deploy to every tenant DB | extensibility/feature-toggles |

## CAP-SEC — Security (18)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Explicit authorization on every exposed service | Critical | REQ | B | CDS services have no access control by default | guides/security/authorization |
| 2 | No mocked/dummy auth in production | Critical | REQ | B | Production binds IAS or XSUAA | node.js/authentication, java/security |
| 3 | Keep deny-by-default (`restrict_all_services` untouched) | Critical | REQ | N | ⏱ Implicit `@requires:'authenticated-user'` in production | node.js/authentication |
| 4 | Java strict authentication mode (`model-strict`) | High | ORG (SAP option) | J | Harden beyond default `model-relaxed` | java/security |
| 5 | Java: verify security deps AND binding present | Critical | REQ | J | Auto-enforcement needs both | java/security |
| 6 | IAS-first for new projects | High | REC | B | ⏱ XSUAA legacy; cross-consumption bridge | guides/security/authentication |
| 7 | IAS tokens carry no scopes — pair with AMS | Critical | REQ | B | Authorization must be explicitly provided | node.js/authentication, cap-users |
| 8 | Business roles; exact scope↔role name mapping; regenerate xs-security.json | High | REQ | B | `cds compile srv --to xsuaa` | guides/security/cap-users |
| 9 | Technical roles never in business roles; `internal-user` isolation | Critical | REQ | B | `cds.Subscriber`, `mtcallback`, … | data-protection, cap-users |
| 10 | Instance-based authorization declaratively (`@restrict…where`) | High | REC | B | `$user`, attributes, `exists`; handlers preserve filters | guides/security/authorization |
| 11 | Composition/association exposure review | High | REQ (documented gaps) | B | Java: composition children unchecked; Node: expands/deep inserts unchecked | guides/security/authorization |
| 12 | Declarative validation on externally writable elements | High | REC | B | `@mandatory`/`@assert.*` before handler validation | guides/services/constraints |
| 13 | No user input in query structure | Critical | REQ | B | Positive-list entity/column names; values auto-parameterized | security/data-protection |
| 14 | DoS limits configured (`$batch`, `$expand`, pagination, rate limits) | Medium | REQ (responsibility) | B | Node `$expand` limit needs custom handler (ORG pattern) | security/data-protection |
| 15 | App Router is not a security boundary | Critical | REQ | B | Backends authenticate independently; test unauthenticated rejection | security/authentication, data-protection |
| 16 | Secure logging (INFO default, no secrets/PII, Java escaping) | Medium | REC/REQ | B | CRLF-safe API (N); manual escaping (J) | security/data-protection |
| 17 | `@PersonalData` on all personal data | High | REC | B | Drives audit/PDM/DRM automation | guides/security/data-privacy |
| 18 | No secrets on disk in development (`cds bind`, localhost only, no prod data) | High | REC | B | Pointers not credentials | security/overview, tools/cds-bind |

## CAP-PRIV — Data privacy & audit (4)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Complete `@PersonalData` semantics (EntitySemantics + DataSubjectID) | High | REQ | B | Selective `IsPotentiallySensitive` (audit-on-read cost) | security/dpp-annotations |
| 2 | Privacy annotations in dedicated file | Medium | REC | B | e.g. `srv/data-privacy.cds` | dpp-annotations |
| 3 | Audit logging plugin + SAP Audit Log service in production | High | REC/REQ | B | Outbox delivery kept on; console only in dev | dpp-audit-logging |
| 4 | PDM services flat + role-protected | High | REQ | B | `@requires:'PersonalDataManagerUser'` for PDM instance only | dpp-pdm |

## CAP-ERR — Error handling (7)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | `req.error`/`req.reject` for client errors, not raw throws | High | REC | N | Protocol-correct error rendering | node.js/events |
| 2 | Never disable 5xx sanitization in production | Critical | REQ | N | No `err.$sanitize = false` | node.js/events |
| 3 | Fail loudly; don't catch what you can't handle | High | REC | B | "Let it crash"; augment-and-rethrow | node.js/best-practices |
| 4 | `srv.on('error')` handlers synchronous only | High | REQ | N | No async/Promises | node.js/core-services |
| 5 | `ServiceException` + `ErrorStatuses`; `Messages` for multi-condition validation | High | REC | J | ⏱ Java 5: `preferServiceException` default true | java/…/indicating-errors |
| 6 | Localized error codes/keys with targets | Medium | REC | B | i18n bundles; Fiori field targets | events, indicating-errors |
| 7 | Validate user input embedded in messages | Critical | REQ | B | Injection prevention | events, indicating-errors |

## CAP-LOG — Logging & observability (6)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | `cds.log`/SLF4J only; parameterized, no concat, no console.log | High | REC | B | Guard debug hot paths | node.js/cds-log, java observability |
| 2 | JSON structured logs in production | High | REC | B | ⏱ Node default since cds 7.5; Java cf-java-logging-support | cds-log, observability |
| 3 | Preserve correlation-ID propagation | Medium | REC | B | Framework-managed; don't strip | cds-log, observability |
| 4 | Per-component log levels; keep masking on | Medium | REC | B | No global DEBUG in production | cds-log, observability |
| 5 | OpenTelemetry via SAP tooling (`@cap-js/telemetry` / buildpack agent) | Medium | REC | B | ⏱ Telemetry v2 with cds 10 | plugins/, observability |
| 6 | Expose only health endpoints publicly | Critical | REQ | B | ⏱ Node `/health` since cds 7.8; other actuators local/JMX | observability, best-practices |

## CAP-TEST — Testing (7)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Bootstrap with `cds.test()` before touching `cds.env` | High | REQ | N | ⏱ `@cap-js/cds-test` (cds 8+) | node.js/cds-test |
| 2 | In-memory SQLite + `data.reset()` isolation | High | REC | N | Per-test isolation | cds-test |
| 3 | Assert stable codes, not message text/snapshots | Medium | REC | N | `containSubset`, minimal assertions | cds-test |
| 4 | Runner-portable tests | Low | REC | N | ⏱ Vitest in, Jest being abandoned (cds 10) | cds-test, jun26 |
| 5 | Java layered testing (unit/Mockito → service/CQN → `@SpringBootTest`+MockMvc) | High | REC | J | ⏱ H2 pinned 2.3.x; MTX needs hybrid tests | java/…/testing |
| 6 | Mock users for authenticated test flows | Medium | REC | B | `@MockUser` / cds.test auth | testing, cds-test |
| 7 | Hybrid tests via `cds bind --exec` for cloud-only features | Medium | REC | B | HANA/MTX scenarios | cds-test, tools/cds-bind |

## CAP-PERF — Performance & scalability (9)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Intentional pagination limits (`@cds.query.limit`); never disable | High | REC | B | Hard default 1,000 | guides/services/served-ootb |
| 2 | Reliable pagination for concurrent OData V4 collections | Medium | REC | B | Opt-in; simple `$orderby` only | served-ootb |
| 3 | Associations + `$expand` over static JOIN views | High | REC | B | JOINs execute only on expansion | guides/databases/performance |
| 4 | No live-calculated fields in WHERE/ORDER BY/JOIN | High | REC | B | Bypass indexes; pre-calculate on write | databases/performance |
| 5 | Avoid UNIONs; remodel polymorphism | Medium | REC | B | Type discriminator + compositions/aspects | databases/performance |
| 6 | Decimal/Int64 arithmetic in the database | Medium | REC | N | JS Number loses precision | node.js/best-practices |
| 7 | `srv.foreach` for large result sets | Medium | REC | N | Avoid full materialization | node.js/core-services |
| 8 | Single-field primary keys | Low | REC | B | Multi-field keys hurt JOINs | guides/domain/ |
| 9 | Composition-tree size awareness (drafts copy whole documents) | Medium | REC | B | Associations for very large child sets | databases/performance |

## CAP-DEP — Deployment (6)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | MTA-based CF deployment (`cds add mta`, `mbt build`, `cds up`) | High | REC | B | ⏱ `cds up` since Mar 2025 | guides/deploy/to-cf |
| 2 | Production facets before first deploy (`cds add hana,xsuaa`) | Critical | REQ | B | Never SQLite/H2/mocked auth in prod | to-cf |
| 3 | Landscape config in `.mtaext` extension descriptors | Medium | REC | B | Not in base mta.yaml | to-cf |
| 4 | Kyma: `cds add kyma` Helm + Cloud Native Buildpacks | High | REC | B | Customize values.yaml only | guides/deploy/to-kyma |
| 5 | `cds bind` for hybrid work — no materialized credentials | High | REC | B | `.cdsrc-private.json` pointers | tools/cds-bind |
| 6 | Read-only pull secrets for private registries | Medium | REC | B | Limited technical users | to-kyma |

## CAP-CICD — CI/CD (3)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Pipelines scaffolded via `cds add github-actions` (or SAP CI/CD / Piper) | High | REC | B | ⏱ GitHub Actions is the documented default | guides/deploy/cicd |
| 2 | Production deploys via protected environment + tagged releases | Medium | REC | B | No laptop deploys | cicd |
| 3 | Cloud-backed integration tests via `cds bind --exec` in CI | Medium | REC | B | No persisted credentials | tools/cds-bind |

## CAP-VER — Dependency & version management (9)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Freeze dependencies via lockfile before deploying | Critical | REQ | N | `npm install --package-lock-only` | guides/deploy/to-cf |
| 2 | Regular dependency refresh (Dependabot/Renovate, one-by-one) | High | REC | B | ORG cadence needed | to-cf |
| 3 | Stay on Active CAP major; plan within maintenance window | Critical | REQ (policy) | B | ⏱ Active → Maintenance (≤12 mo) → EOL | releases/schedule |
| 4 | Latest minors monthly, patches ASAP (Java) | High | REC | J | SAP consumption policy | java/versions |
| 5 | Current runtime baselines (Node ≥ 22, JDK ≥ 21) | Critical | REQ | B | ⏱ As of cds 10 / Java 5 (June 2026) | releases/2026/jun26 |
| 6 | Never mix CAP package major lines | Critical | REQ | N | ⏱ cds 10 ↔ dk 10 ↔ compiler 7 ↔ mtxs 4 ↔ @cap-js 3 | jun26 |
| 7 | Official migration tooling at majors (`cds upgrade` alpha + manual review; OpenRewrite) | High | REC | B | ⏱ False positives documented | node.js/upgrading, java/migration |
| 8 | `hdbtable` deploy format (no `hdbcds`) | High | REQ | B | ⏱ Unusable since cds 9 | releases/2025/may25 |
| 9 | Outbox drain before cds 8→10 upgrade (or via v9) | High | REQ | N | ⏱ Double-processing hazard | guides/events/event-queues |

## CAP-OPS — Production readiness & operations (4)

| # | Title | Sev | Auth | RT | Statement (short) | Source |
|---|---|---|---|---|---|---|
| 1 | Health probes wired in deployment descriptors | High | REC | B | ⏱ Node `/health`; Java actuator liveness/readiness; CF readiness tooling gap | guides/deploy/health-checks |
| 2 | Real production UI entry point (approuter/portal/workzone) | Medium | REC | B | Dev index page unavailable in prod | to-cf |
| 3 | MCP exposure governed (beta; read-only; no injection/rate/audit protection) | High | REQ (documented warnings) | B | ⏱ Beta June 2026; not for SAP Application APIs | guides/protocols/mcp |
| 4 | Deployment topology by configuration, not code | Medium | REC | B | Monolith ↔ microservices | guides/deploy/microservices |

---

## Promotion order recommendation (AI-REC)

For Phase 2, author categories in this order — highest review value first:
1. **CAP-SEC** (18) and **CAP-MT** (5) — most Critical rules, most REQ authority
2. **CAP-DB**, **CAP-TXN**, **CAP-EVT** — data-integrity Criticals (injection, outbox, idempotency)
3. **CAP-ARCH**, **CAP-SRV**, **CAP-CDS**, **CAP-LOGIC** — the architecture/development core
4. **CAP-VER**, **CAP-DEP**, **CAP-OPS**, **CAP-CICD** — gate M8/M9
5. **CAP-TEST**, **CAP-ERR**, **CAP-LOG**, **CAP-PERF**, **CAP-INT**, **CAP-EXT**, **CAP-PRIV** — completes the catalog
