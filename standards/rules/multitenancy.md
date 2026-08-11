# CAP-MT — Multitenancy

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-34, G-35 in [research-gaps.md](../../references/research-gaps.md).

**Applicability precondition for the whole category:** these rules apply to **multitenant** CAP applications (SaaS apps serving multiple subscriber tenants). For single-tenant applications, reviewers mark them NOT APPLICABLE with the project profile as evidence. Note: multitenancy is **not supported on PostgreSQL** per current SAP docs — MT projects target SAP HANA Cloud.

**Rules:** 6 active (2 Critical, 4 High). All SAP references verified against official CAP documentation on **2026-08-11**, including a targeted re-verification pass of every SAP-REQ claim.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-MT-001 | Use streamlined MTX (`@sap/cds-mtxs`) only | High | SAP-REQ | Both |
| CAP-MT-002 | Deploy and scale the MTX sidecar correctly | High | SAP-REQ (Java) / SAP-REC (Node.js) | Both |
| CAP-MT-003 | Preserve strict tenant isolation in persistence and custom code | Critical | SAP-REQ | Both |
| CAP-MT-004 | Tenant lifecycle handlers are idempotent | High | SAP-REQ | Both |
| CAP-MT-005 | Upgrade all tenants before a new model version serves traffic | Critical | SAP-REQ | Both |
| CAP-MT-006 | Establish explicit tenant context for background and asynchronous work | High | SAP-REC | Both |

---

## CAP-MT-001 — Use streamlined MTX (`@sap/cds-mtxs`) only

| Field | Value |
|---|---|
| **Rule ID** | CAP-MT-001 |
| **Title** | Use streamlined MTX (`@sap/cds-mtxs`) only |
| **Category** | Multitenancy |
| **Severity** | High |
| **Authority** | SAP-REQ (old MTX in maintenance mode; supported only up to CAP Java 2.x) |
| **Applicability** | All multitenant projects |
| **Runtime** | Both |
| **CAP version** | ⏱ Version-defining rule: streamlined MTX (`@sap/cds-mtxs`) is the only current line (4.x with cds 10); `@sap/cds-mtx` / CAP Java "classic" MT ended with CAP Java 2.x |
| **Status** | Active |
| **Related rules** | CAP-MT-002, CAP-MT-005; CAP-VER category (Active-major policy) |
| **Last verified** | 2026-08-11 |

### Rule statement
Multitenancy MUST be implemented with streamlined MTX — `cds add multitenancy`, `@sap/cds-mtxs` (Node.js), `cds-feature-mt` with the MTX sidecar (Java). The legacy `@sap/cds-mtx` package and CAP Java classic multitenancy MUST NOT be used in new projects; existing users MUST have a migration plan following SAP's old-MTX migration guide.

### Rationale
SAP documents old MTX as maintenance-only and supported only until CAP Java 2.x; current CAP majors (Java 5 / cds 10) require streamlined MTX, and migration tooling from classic was removed in the cds 10 wave. **High (downgraded from candidate Critical):** the legacy stack still functions where deployed — the risk is an unsupported security-fix and upgrade path in a multitenant SaaS, which blocks a milestone but is not an immediate exposure.

### Evidence expected in code
`@sap/cds-mtxs` in dependencies (app or `mtx/sidecar` package.json); Java: `cds-feature-mt` in `pom.xml` plus the sidecar project; absence of `@sap/cds-mtx`.

### Detection guidance
1. Search all `package.json` files for `@sap/cds-mtx` (exact, not `-mtxs`) → any occurrence → FAIL.
2. Verify `@sap/cds-mtxs` present (root or `mtx/sidecar`) and `cds.requires.multitenancy` configured; Java: `cds-feature-mt` in `pom.xml`.
3. Check the `@sap/cds-mtxs` major line matches the `@sap/cds` major per the [version register](../../docs/version-management.md) (e.g., mtxs 4.x with cds 10) → mismatch → FAIL (cross-report to CAP-VER).
4. For legacy findings, check for a documented migration plan; absent → FAIL without mitigation note.

### Good example
```jsonc
// mtx/sidecar/package.json
{ "dependencies": { "@sap/cds": "^10", "@sap/cds-mtxs": "^4" },
  "cds": { "profile": "mtx-sidecar" } }
```

### Bad example
```jsonc
// package.json — deprecated MTX classic, unsupported on current majors
{ "dependencies": { "@sap/cds-mtx": "^2" } }
```

### Exception guidance
Only a documented, time-boxed migration period for an existing application (per SAP's migration guide), recorded as an exception with an end date.

### SAP reference
- https://cap.cloud.sap/docs/guides/multitenancy/ (streamlined MTX setup)
- https://cap.cloud.sap/docs/guides/multitenancy/old-mtx-migration (legacy status and migration)
- https://cap.cloud.sap/docs/java/multitenancy-classic (classic supported only ≤ CAP Java 2.x)

---

## CAP-MT-002 — Deploy and scale the MTX sidecar correctly

| Field | Value |
|---|---|
| **Rule ID** | CAP-MT-002 |
| **Title** | Deploy and scale the MTX sidecar correctly |
| **Category** | Multitenancy |
| **Severity** | High |
| **Authority** | SAP-REQ for Java ("Java-based projects even require such a sidecar"); SAP-REC for Node.js production |
| **Applicability** | All multitenant projects |
| **Runtime** | Both (the sidecar itself is always Node.js) |
| **CAP version** | Current streamlined MTX; single-instance-first guidance per current multitenancy guide |
| **Status** | Active |
| **Related rules** | CAP-MT-001, CAP-MT-005 |
| **Last verified** | 2026-08-11 |

### Rule statement
CAP Java multitenant applications MUST deploy the Node.js MTX sidecar (MTX services are implemented in Node.js). Node.js multitenant applications SHOULD run MTX as a separate sidecar in production rather than embedded, so resource-intensive subscribe/upgrade operations scale independently of request traffic. The **initial** rollout MUST deploy the sidecar with a single instance to avoid conflicts while the `t0` metadata tenant is created; scaling out afterwards is fine.

### Rationale
SAP documents all three elements: the Java sidecar requirement, the recommendation to separate MTX for independent scaling, and the single-instance-first deployment to avoid `t0` creation conflicts. Violations produce broken subscription operations (Java without sidecar: no MT at all; parallel first-rollout instances: corrupted t0 bootstrap; embedded MTX under load: subscription/upgrade starving business traffic). **High:** operational integrity of the tenant lifecycle, not direct data exposure.

### Implementation guidance
- `cds add multitenancy` scaffolds `mtx/sidecar` — keep it a separate module in `mta.yaml`/Helm chart.
- Express the single-instance-first constraint in the deployment artifact (initial `instances: 1` with a comment, or a documented first-rollout step in the runbook).

### Evidence expected in code
`mtx/sidecar` module present and wired in `mta.yaml`/chart (Java: mandatory; Node.js production: expected or a documented embedded-MTX decision); initial-deployment instance constraint visible in descriptor or runbook.

### Detection guidance
1. Confirm the project is multitenant (per category precondition).
2. Java: verify a sidecar module exists (`mtx/sidecar` with `@sap/cds-mtxs`) and is a deployable module in `mta.yaml`/chart → missing → FAIL.
3. Node.js: determine embedded vs sidecar MTX; embedded in production → FAIL unless a documented decision accepts the scaling trade-off.
4. Check the sidecar module's instance configuration and the deployment runbook for the single-instance-first rollout; no trace anywhere → FAIL this element (report as first-rollout risk; for long-deployed systems mark NOT ASSESSABLE — historical fact).
5. Report with file:line into `mta.yaml`/chart values.

### Good example
```yaml
# mta.yaml
modules:
  - name: bookshop-mtx
    type: nodejs
    path: mtx/sidecar
    parameters:
      instances: 1   # initial rollout: keep 1 until t0 exists (see runbook §3)
```

### Bad example
```yaml
# mta.yaml — CAP Java MT app with no sidecar module at all;
# subscription callbacks have nothing to call
modules:
  - name: bookshop-srv
    type: java
    path: srv
```

### Exception guidance
Node.js apps with trivially low tenant counts may run embedded MTX with a documented decision (accepted scaling impact). No exception for Java.

### SAP reference
- https://cap.cloud.sap/docs/guides/multitenancy/ (sidecar; Java requirement; single instance for initial rollout)
- https://cap.cloud.sap/docs/guides/multitenancy/mtxs (MTX services implemented in Node.js; sidecar consumption)

---

## CAP-MT-003 — Preserve strict tenant isolation in persistence and custom code

| Field | Value |
|---|---|
| **Rule ID** | CAP-MT-003 |
| **Title** | Preserve strict tenant isolation in persistence and custom code |
| **Category** | Multitenancy |
| **Severity** | Critical |
| **Authority** | SAP-REQ |
| **Applicability** | All multitenant projects; every piece of custom code in them |
| **Runtime** | Both |
| **CAP version** | All currently supported versions; MT not supported on PostgreSQL (current docs) |
| **Status** | Active |
| **Related rules** | CAP-SEC-002, CAP-SEC-013, CAP-MT-006 |
| **Last verified** | 2026-08-11 |

### Rule statement
Tenant data MUST remain strictly isolated: rely on CAP's per-tenant persistence (a database/HDI container per tenant, provisioned by MTX) and never bypass tenant-aware data access. Custom code MUST NOT undermine isolation: no static mutable state holding request/tenant data (Java), no data captured in module-level variables or service-implementation closures (Node.js), no hand-built cross-tenant queries or hardcoded tenant IDs, no caching keyed without the tenant. Tenant isolation MUST be covered by tests (e.g., mock tenants `t1`/`t2` locally, asserting one tenant never sees the other's data).

### Rationale
SAP defines multitenancy as serving multiple tenants "while strictly isolating the tenants' data", provisions a persistent tenant database per tenant, and instructs: "Make sure that custom code doesn't break tenant data isolation" — with the CAP Java runtime itself refraining from static mutable heap state and the Node.js guidance to keep non-static variables out of service-implementation closures. The framework isolates; only custom code can break it. **Critical justification:** violation is direct cross-tenant data leakage — the defining catastrophic failure of a SaaS application.

### Implementation guidance
- Access data only through the request's transactional context (`cds.context`-bound service instances / handler-provided `req`; Java: the event context's service instances) — never module-level `db` handles capturing state.
- Any cache must be tenant-scoped (key includes `req.tenant` / `EventContext.getUserInfo().getTenant()`), with eviction on unsubscribe.
- Write an isolation test early: subscribe two mock tenants, write as `t1`, assert invisibility as `t2` (SAP's local sidecar + mock-user tenants support this).

### Evidence expected in code
No module-level/static mutable state carrying business data in `srv/**`; caches keyed by tenant; no `tenant`-parameter overrides pointing at fixed tenant IDs; isolation tests present.

### Detection guidance
1. Node.js: inspect `srv/**/*.js|ts` for module-scope `let/var` holding data, closures in `cds.service.impl`/`init()` capturing per-request data, and caches (`new Map()` at module scope) without tenant in the key → each → FAIL.
2. Java: search for mutable `static` fields in handler/service classes (`static` non-final collections, singletons caching entity data) → FAIL.
3. Search for hardcoded tenant IDs and explicit tenant switches (`cds.tx({ tenant: '<fixed>' })`, `systemUser("<fixed>")`) outside documented provider-side administration code → FAIL unless justified per CAP-MT-006.
4. Verify persistence configuration relies on MTX-provisioned per-tenant HDI/DB (no shared-schema workarounds, no discriminator-column tenancy) → deviation → FAIL.
5. Locate tenant-isolation tests; absent → FAIL the test element (report as missing isolation evidence).
6. Report per finding with file:line.

### Good example
```js
// srv/catalog-service.js — per-tenant cache, context-bound access
const caches = new Map();                       // key: tenant
srv.on('READ', 'Top10', async req => {
  const key = req.tenant;
  if (!caches.has(key)) caches.set(key, await computeTop10(srv));
  return caches.get(key);
});
```

### Bad example
```js
// module-level cache shared across ALL tenants — t2 gets t1's data
let top10;
srv.on('READ', 'Top10', async () => top10 ??= await computeTop10(srv));
```

### Exception guidance
Deliberately tenant-independent data (static configuration, provider-owned content) may be cached globally — documented at the cache site. Provider-side administrative cross-tenant operations are governed by CAP-MT-006, never ad hoc.

### SAP reference
- https://cap.cloud.sap/docs/guides/multitenancy/ (strict isolation; per-tenant database provisioning)
- https://cap.cloud.sap/docs/guides/security/data-protection ("Make sure that custom code doesn't break tenant data isolation"; static-state and closure guidance)

---

## CAP-MT-004 — Tenant lifecycle handlers are idempotent

| Field | Value |
|---|---|
| **Rule ID** | CAP-MT-004 |
| **Title** | Tenant lifecycle handlers are idempotent |
| **Category** | Multitenancy |
| **Severity** | High |
| **Authority** | SAP-REQ |
| **Applicability** | Multitenant projects with custom handlers on tenant lifecycle events (subscribe/unsubscribe/upgrade/dependencies) |
| **Runtime** | Both (Node.js MTX hooks; Java `DeploymentService` events) |
| **CAP version** | Streamlined MTX (`cds.xt.DeploymentService` / CAP Java DeploymentService events) |
| **Status** | Active |
| **Related rules** | CAP-MT-002, CAP-MT-005, CAP-SEC-009 (technical roles for lifecycle endpoints) |
| **Last verified** | 2026-08-11 |

### Rule statement
Custom handlers for tenant lifecycle events MUST be idempotent: SAP documents that subscription may trigger multiple times for the same tenant (the SaaS registry uses the same endpoint for initial subscription and later updates), so `subscribe` handlers MUST tolerate repeated invocation without duplicating artifacts or failing; `unsubscribe` handlers MUST handle already-removed tenants gracefully; `upgrade` handlers MUST be safely re-runnable.

### Rationale
SAP states custom subscribe handlers "must therefore be idempotent and able to handle multiple calls for the same tenant". Non-idempotent lifecycle code corrupts onboarding: duplicate provisioning artifacts, failed re-subscriptions, stuck tenants — discovered exactly when a customer subscribes. **High:** tenant lifecycle integrity; does not by itself expose data (that's CAP-MT-003 / CAP-SEC-009 territory).

### Implementation guidance
- Make every provisioning step check-then-act or upsert-style (create-if-absent), including external artifacts (queues, destinations, schedules).
- Treat unsubscribe cleanup as "ensure absent", not "delete once".
- Test lifecycle handlers by invoking subscribe twice for the same tenant in the local sidecar setup.

### Evidence expected in code
Lifecycle handlers (Node.js: handlers on `cds.xt.DeploymentService`/SaaS provisioning events, typically in `mtx/sidecar` or server.js; Java: `@Before`/`@On`/`@After` on subscribe/unsubscribe/upgrade event contexts) written check-then-act; a repeated-subscription test.

### Detection guidance
1. Locate custom lifecycle handlers: search for `cds.xt.DeploymentService`, `'subscribe'`/`'unsubscribe'`/`'upgrade'` handler registrations (Node.js) and `SubscribeEventContext`/`UnsubscribeEventContext`/DeploymentService handler classes (Java). None → NOT APPLICABLE.
2. For each handler, inspect side effects: unconditional `INSERT`/create calls, external resource creation without existence checks, counters/append operations → each non-idempotent effect → FAIL with file:line.
3. Check unsubscribe handlers for hard failures on missing artifacts (unguarded deletes/reads) → FAIL.
4. Look for a test invoking subscribe twice for one tenant; absent → report missing evidence element.

### Good example
```js
// mtx/sidecar/srv/provisioning.js
const { DeploymentService } = cds.services['cds.xt.DeploymentService'] ? cds.services : await cds.connect.to('cds.xt.DeploymentService')
DeploymentService.after('subscribe', async ({ tenant }) => {
  await ensureEventQueue(tenant);        // creates only if absent
  await upsertTenantConfig(tenant, defaults);
});
```

### Bad example
```js
DeploymentService.after('subscribe', async ({ tenant }) => {
  await createEventQueue(tenant);        // throws QueueAlreadyExists on re-subscribe
  await INSERT.into(TenantConfig).entries({ tenant, ...defaults }); // duplicates rows
});
```

### Exception guidance
None on idempotency itself. Steps that are inherently once-only (e.g., issuing a welcome notification) must internally record-and-skip on repeat rather than relying on single delivery.

### SAP reference
- https://cap.cloud.sap/docs/guides/multitenancy/mtxs (subscribe handlers "must therefore be idempotent and able to handle multiple calls for the same tenant")
- https://cap.cloud.sap/docs/java/multitenancy (DeploymentService subscribe/unsubscribe/upgrade events)

---

## CAP-MT-005 — Upgrade all tenants before a new model version serves traffic

| Field | Value |
|---|---|
| **Rule ID** | CAP-MT-005 |
| **Title** | Upgrade all tenants before a new model version serves traffic |
| **Category** | Multitenancy |
| **Severity** | Critical |
| **Authority** | SAP-REQ |
| **Applicability** | Every deployment of a multitenant app that changes the CDS model (or database-relevant artifacts) |
| **Runtime** | Both |
| **CAP version** | Streamlined MTX; Java main class `com.sap.cds.framework.spring.utils.Deploy` per current docs; HANA upgrades use schema evolution, SQLite dev upgrades drop-create (data loss — dev only) |
| **Status** | Active |
| **Related rules** | CAP-MT-002, CAP-MT-004; orchestration specifics (zero-downtime, batching, canary tenants) are ORG — gap G-34 |
| **Last verified** | 2026-08-11 |

### Rule statement
Deployments that change the CDS model MUST run the tenant upgrade for **all** tenants **before** the new application version serves traffic: Java — run the `com.sap.cds.framework.spring.utils.Deploy` main class (with the MTX sidecar running) before starting the new version; Node.js — run the MTX upgrade for all tenants (`cds-mtx upgrade '*'` via deployment hook, CF task, or Kubernetes job) ahead of traffic switchover. The upgrade step MUST be part of the deployment automation, not a manual afterthought.

### Rationale
SAP documents the ordering and its reason: running the update for all tenants before starting the new version "prevents new application code to access database artifacts that aren't yet deployed", and warns the sidecar must be running during the upgrade. New code against un-upgraded tenant schemas fails at runtime for every affected tenant — and write paths against half-matching schemas risk corrupt data. **Critical justification:** violation degrades or corrupts **all tenants simultaneously** on every model-changing release — production integrity failure at SaaS scale.

### Implementation guidance
- Encode the ordering in the deployment artifact (MTA deploy hooks/CF task before route switch; Helm pre-upgrade job) rather than a runbook step someone must remember.
- Batching, canary tenants, and zero-downtime strategies are ORG policy (G-34) — this rule fixes only the invariant: upgrade completes before new code serves a tenant.

### Evidence expected in code
A tenant-upgrade step in deployment automation (`mta.yaml` deploy hooks/tasks, pipeline stage, Helm pre-upgrade job) ordered before app start/traffic switch; runbook documenting the sequence.

### Detection guidance
1. Confirm the deployment changes models over time (any live MT app) — the rule applies to the deployment *process*.
2. Inspect `mta.yaml` (hooks/tasks), CI pipeline definitions, and Helm chart (pre-upgrade jobs) for a tenant upgrade invocation (`cds-mtx upgrade`/MTX upgrade API/`Deploy` main class).
3. Verify ordering: the upgrade step runs before the new version receives traffic (hook phase / job ordering / pipeline stage sequence).
4. No automated upgrade step anywhere → FAIL. Present but unordered/parallel to app start → FAIL with the descriptor location.
5. Check the sidecar is available during upgrade (not stopped by the same deployment step) → violation → FAIL.
6. Runtime-only aspects of past deployments → NOT ASSESSABLE; name the pipeline-run evidence needed.

### Good example
```yaml
# mta.yaml — upgrade all tenants as a deploy task before app restart
modules:
  - name: bookshop-mtx
    type: nodejs
    path: mtx/sidecar
    parameters:
      tasks:
        - name: upgrade-tenants
          command: cds-mtx upgrade '*'
```

### Bad example
```text
Pipeline pushes the new Java app version directly; tenant upgrade is a
wiki instruction "run Deploy main afterwards" — every model-changing
release breaks all tenants until someone remembers.
```

### Exception guidance
Deployments provably not touching the model/database artifacts (config-only, UI-only) may skip the upgrade step when the pipeline detects this automatically; the detection logic must itself be reviewable. No exception for model-changing releases.

### SAP reference
- https://cap.cloud.sap/docs/java/multitenancy (run upgrades before starting the new version; sidecar must be running; `Deploy` main class)
- https://cap.cloud.sap/docs/guides/multitenancy/ (upgrading tenant databases via MTX)

---

## CAP-MT-006 — Establish explicit tenant context for background and asynchronous work

| Field | Value |
|---|---|
| **Rule ID** | CAP-MT-006 |
| **Title** | Establish explicit tenant context for background and asynchronous work |
| **Category** | Multitenancy |
| **Severity** | High |
| **Authority** | SAP-REC |
| **Applicability** | Multitenant projects running work outside a request context: schedulers, startup jobs, queue processors, custom threads/async continuations |
| **Runtime** | Both |
| **CAP version** | Current docs: Java `RequestContextRunner` API (`runtime.requestContext()`, `systemUser(tenant)`, `systemUserProvider()`), `TenantProviderService`; Node.js `cds.context`/`cds.spawn`/`cds.tx` context handling; queued/outbox messages restore tenant + user ID |
| **Status** | Active |
| **Related rules** | CAP-MT-003, CAP-SEC-009; CAP-EVT-002/-003/-004 (queue semantics) |
| **Last verified** | 2026-08-11 |

### Rule statement
Work executing outside an inbound request MUST run in an explicitly established tenant context using the framework's facilities — Java: the `RequestContextRunner` API (`runtime.requestContext().systemUser(tenant).run(...)`, `systemUserProvider()` for provider-account work, `TenantProviderService` to enumerate tenants); Node.js: framework-propagated `cds.context` (e.g., via `cds.spawn`/`cds.tx` with an explicit `tenant` where context isn't inherited). Named-user contexts MUST NOT be propagated into asynchronous processing (outbox/messaging) — SAP directs using technical users for such scenarios. Iterating "all tenants" uses the framework tenant list, never a hardcoded set.

### Rationale
SAP documents the `RequestContextRunner` API for opening request contexts with defined user/tenant, and warns: "Don't try to propagate named user context in asynchronous requests… Instead, use technical users for such scenarios" (inconsistencies or security issues otherwise). Background work with an absent or wrong tenant context reads or writes the wrong tenant's database, or fails obscurely. **High:** the framework defaults are safe for request-triggered flows and the queue restores tenant context; this rule targets the custom-scheduler/thread edge where mistakes land in CAP-MT-003 territory.

### Implementation guidance
- Java scheduled jobs: enumerate tenants via `TenantProviderService`, then `requestContext().systemUser(tenant).run(ctx -> …)` per tenant.
- Node.js: prefer `cds.spawn` (inherits/sets context) over bare `setInterval`/`setTimeout`; pass `{ tenant }` explicitly for work not rooted in a request; always `await` async operations inside.
- Let the persistent queue/outbox carry deferred work where possible — it restores tenant (and user ID) on processing; carry additional claims in the payload, not by impersonating the named user.

### Evidence expected in code
Scheduler/startup/async code wrapped in framework context APIs with explicit tenant; technical users (not propagated named users) for async processing; tenant enumeration via framework services.

### Detection guidance
1. Locate out-of-request execution: Java `@Scheduled`/`ApplicationRunner`/custom `Thread`/`CompletableFuture` usage in `srv/src/main/java/**`; Node.js `setInterval|setTimeout|cron` and bare async fire-and-forget in `srv/**`.
2. For each site, verify a framework context wrapper with explicit tenant (Java `requestContext()...run`; Node `cds.spawn`/`cds.tx({ tenant })` or documented context inheritance) → unwrapped DB/service access → FAIL with file:line.
3. Search for named-user propagation into async paths (storing `req.user` for later impersonation, forwarding user JWTs into queued jobs) → FAIL (use technical users).
4. Check "all tenants" loops use `TenantProviderService`/MTX tenant listing, not hardcoded arrays → FAIL otherwise.
5. Cross-check queue/outbox usage: handlers relying on caller roles at processing time (queue restores only user ID, runs privileged) → report under CAP-EVT-004; note here as observation.

### Good example
```java
// Java — nightly job per tenant, system user, framework-managed context
List<String> tenants = tenantProvider.readTenants();
for (String tenant : tenants) {
  runtime.requestContext().systemUser(tenant).run(ctx ->
      recalcService.run(Update.entity(RANKINGS).data(refresh())));
}
```

### Bad example
```java
// Java — @Scheduled method calls PersistenceService with NO request
// context: no tenant, wrong or failing DB access at runtime
@Scheduled(cron = "0 0 2 * * *")
void nightly() { db.run(Update.entity(RANKINGS).data(refresh())); }
```

### Exception guidance
Provider-account work that is genuinely tenant-independent (e.g., reading provider-own configuration) uses the provider context (`systemUserProvider()`) — documented at the call site. Anything touching subscriber data has no exception.

### SAP reference
- https://cap.cloud.sap/docs/java/multitenancy (`RequestContextRunner`, `systemUser(tenant)`, `TenantProviderService`)
- https://cap.cloud.sap/docs/java/event-handlers/request-contexts (RequestContextRunner API; thread propagation)
- https://cap.cloud.sap/docs/guides/security/cap-users (don't propagate named users to asynchronous requests; use technical users)
- https://cap.cloud.sap/docs/node.js/cds-tx (`cds.context` propagation; `cds.spawn`)
