# CAP-TXN — Transactions

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gap: G-33 in [research-gaps.md](../../references/research-gaps.md).

**Rules:** 6 active (0 Critical, 3 High, 2 Medium, 1 Low). All SAP references verified against official CAP documentation on **2026-08-11** — including corrections where documentation is weaker than commonly assumed (see CAP-TXN-003/-005).

Scope boundaries: tenant context for background work is [CAP-MT-006](multitenancy.md); reliable side effects (events/audit) via the transactional queue are [CAP-EVT-002](events-messaging.md); locking is [CAP-DB-008](data-persistence.md).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-TXN-001 | Rely on managed transactions in event handlers | High | SAP-REC | Both |
| CAP-TXN-002 | Explicit transactions only outside managed contexts — prefer the functional form | Medium | SAP-REC | Node.js |
| CAP-TXN-003 | Don't pass `req` to `cds.tx` — context propagates automatically | Low | SAP-REC | Node.js |
| CAP-TXN-004 | Control CAP Java transactions via the ChangeSet context or Spring transaction management | Medium | SAP-REC | Java |
| CAP-TXN-005 | Never assume distributed atomicity across transactions | High | SAP-REQ | Both |
| CAP-TXN-006 | Await every asynchronous operation in transactional contexts | High | SAP-REQ | Node.js |

---

## CAP-TXN-001 — Rely on managed transactions in event handlers

| Field | Value |
|---|---|
| **Rule ID** | CAP-TXN-001 |
| **Title** | Rely on managed transactions in event handlers |
| **Category** | Transactions |
| **Severity** | High |
| **Authority** | SAP-REC (documented managed-environment stance on both runtimes) |
| **Applicability** | All event/request handlers |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-TXN-002, CAP-TXN-004, CAP-TXN-006, CAP-SRV-002 |
| **Last verified** | 2026-08-11 |

### Rule statement
Inside event handlers, code MUST rely on CAP's managed transaction handling — Node.js: "you don't have to care about transactions, principal propagation, or tenant isolation at all" in managed environments; Java: a ChangeSet Context is automatically opened around every top-level event, with database transactions started lazily on first Persistence Service interaction. Handlers MUST NOT open, commit, or roll back transactions manually, and MUST NOT create their own root transactions for work that belongs to the current request.

### Rationale
The managed transaction is what ties a request's operations, its rollback-on-error semantics (including collected `req.error` rejections rolling back the whole request), its tenant isolation, and its queued events (CAP-EVT-002) into one consistent unit. Manual transaction handling inside handlers splits work out of that unit: partial commits survive request failures, queued events detach from their business data, and error semantics silently change. **High:** violations produce partially committed request state — a data-integrity defect class — though the framework makes the violation hard to do accidentally.

### Implementation guidance
- Run queries through the handler's service/transaction context (`this`/`srv` in Node handlers; the event context's service instances in Java) — they are automatically transactional.
- To make a request fail, throw / `req.reject` — never attempt a manual rollback.
- Work that deliberately must survive request rollback (e.g., always-log attempts) is the documented use case for a *separate* context — see CAP-TXN-002/-004, and record the reason.

### Evidence expected in code
Handlers free of manual transaction APIs; failures signaled via exceptions/`req.error`/`req.reject`; separate-transaction sites documented.

### Detection guidance
1. Search Node handlers (`srv/**/*.js|ts`) for `cds.tx(` inside `before/on/after` handler bodies → each occurrence → check justification (deliberate separate transaction, documented) → undocumented → FAIL.
2. Search Java handlers for manual transaction APIs: `TransactionTemplate`/`PlatformTransactionManager` usage or `@Transactional(propagation = REQUIRES_NEW)` inside handler classes → same justification check → undocumented → FAIL.
3. Search for manual commit/rollback calls (`tx.commit()`, `tx.rollback()`, `connection.commit()`) in handler paths → FAIL.
4. Verify failure paths use `req.reject`/`req.error`/`ServiceException` (cross-check future CAP-ERR rules), not hand-managed rollback.
5. Report per site with file:line.

### Good example
```js
srv.on('submitOrder', async req => {
  // both operations ride the SAME managed request transaction —
  // if reduceStock throws, the order insert rolls back too
  await INSERT.into(Orders).entries(newOrder(req.data));
  await reduceStock(req.data.book_ID, req.data.quantity);
});
```

### Bad example
```js
srv.on('submitOrder', async req => {
  // separate root transaction: order commits even if the rest of the
  // request fails afterwards — partial state on errors
  await cds.tx(async tx => tx.run(INSERT.into(Orders).entries(newOrder(req.data))));
  await reduceStock(req.data.book_ID, req.data.quantity);
});
```

### Exception guidance
Deliberate decoupling from the request transaction (attempt-logging that must survive rollback, separate technical bookkeeping) is legitimate — with a comment at the site naming the intent, and preferring the transactional queue (CAP-EVT-002) where the goal is reliable side effects rather than rollback-immunity.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-tx ("you don't have to care about transactions…"; managed environments)
- https://cap.cloud.sap/docs/java/event-handlers/changeset-contexts (ChangeSet auto-opened per top-level event; lazy transaction start)

---

## CAP-TXN-002 — Explicit transactions only outside managed contexts — prefer the functional form

| Field | Value |
|---|---|
| **Rule ID** | CAP-TXN-002 |
| **Title** | Explicit transactions only outside managed contexts — prefer the functional form |
| **Category** | Transactions |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | Node.js code running outside request handlers (bootstrap, scripts, background jobs) or needing deliberate separate atomicity |
| **Runtime** | Node.js (Java counterpart: CAP-TXN-004) |
| **CAP version** | All currently supported versions; SQLite dev caveat: parallel transactions deadlock ("Parallel transactions are not allowed") |
| **Status** | Active |
| **Related rules** | CAP-TXN-001, CAP-TXN-006, CAP-MT-006 (tenant context for the same code paths) |
| **Last verified** | 2026-08-11 |

### Rule statement
Explicit `cds.tx()` usage is reserved for non-managed environments (SAP: manual `cds.tx()` is for use "only in non-managed environments" — startup code, scripts, background jobs) and for deliberate multi-operation atomicity. The functional form `cds.tx(async tx => { … })` MUST be preferred: it "creates a new root transaction; executes all nested operations in this transaction; automatically finalizes the transaction with commit or rollback." Hand-managed begin/commit/rollback sequences are prohibited where the functional form suffices. On SQLite (development), parallel explicit transactions MUST be avoided — they deadlock.

### Rationale
The functional form makes commit/rollback structurally correct — no leaked open transactions on exceptions, no forgotten finalization (after finalize, the transaction object is unusable). Hand-managed sequences re-create exactly the error-path bugs the framework eliminates. **Medium:** scoped to unmanaged code paths; the failure mode (leaked/hung transactions, connection exhaustion) is operational and usually loud.

### Evidence expected in code
`cds.tx(async tx => …)` in bootstrap/jobs; no `.begin()`-style or manually finalized transactions where the functional form works; background jobs also establishing tenant context per CAP-MT-006.

### Detection guidance
1. Locate `cds.tx(` usages outside handler bodies (server.js, scripts, scheduled jobs).
2. Functional form → PASS for the form check.
3. Non-functional usage (transaction object obtained and manually committed/rolled back) → verify a reason the functional form couldn't be used → none → FAIL with file:line.
4. Check error paths of any manual form: rollback on all failure paths, no reuse after finalization → gap → FAIL.
5. In multitenant projects, cross-check the same sites against CAP-MT-006 (explicit tenant).
6. Tests running parallel transactions on SQLite → flag the documented deadlock risk (observation).

### Good example
```js
// bootstrap seeding — auto commit/rollback via the functional form
await cds.tx(async tx => {
  await tx.run(INSERT.into(Config).entries(defaults));
  await tx.run(UPDATE(Meta).set({ seeded: true }));
});
```

### Bad example
```js
const tx = cds.tx();                     // manual lifecycle
await tx.run(INSERT.into(Config).entries(defaults));
await tx.commit();                        // no rollback on the throw above —
                                          // leaked transaction on error paths
```

### Exception guidance
Advanced orchestration genuinely needing manual lifecycle control (rare) is acceptable with correct error-path handling and a comment naming why the functional form was insufficient.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-tx (manual `cds.tx()` "only in non-managed environments"; functional form auto-finalizes; SQLite parallel-transaction deadlocks)

---

## CAP-TXN-003 — Don't pass `req` to `cds.tx` — context propagates automatically

| Field | Value |
|---|---|
| **Rule ID** | CAP-TXN-003 |
| **Title** | Don't pass `req` to `cds.tx` — context propagates automatically |
| **Category** | Transactions |
| **Severity** | Low |
| **Authority** | SAP-REC (SAP: "This still works but is not required nor recommended anymore" — not formally deprecated) |
| **Applicability** | Node.js handler code |
| **Runtime** | Node.js |
| **CAP version** | ⏱ Current guidance (continuation-local context propagation); older CAP samples used `cds.tx(req)` — watch for copy-paste from stale material |
| **Status** | Active |
| **Related rules** | CAP-TXN-001, CAP-TXN-002 |
| **Last verified** | 2026-08-11 |

### Rule statement
Handler code SHOULD NOT use the legacy `cds.tx(req)` pattern: the request context (transaction, user, tenant) propagates automatically through CAP's continuation-local context — per SAP, passing `req` "still works but is not required nor recommended anymore". Queries run via the service (`this`, `srv`, connected services) are already transactional in the request's context.

### Rationale
The pattern is legacy boilerplate that obscures the actual model (automatic context propagation) and marks code copied from outdated samples — a maintenance signal, not a defect. **Low:** it still works; the cost is noise and stale idiom. (Correction from the Phase 1 inventory: the docs do not use the word "deprecated" and there is no removal statement — authority is SAP-REC, severity Low.)

### Evidence expected in code
Handlers running queries via services directly; no `cds.tx(req)` in new code.

### Detection guidance
1. Search `srv/**/*.js|ts` for `cds.tx(req` (and `cds.transaction(req`).
2. Each occurrence → FAIL (Low) with file:line and the direct-service replacement.
3. Widespread occurrences → note in the report as a stale-idiom signal (project likely built from old templates — prompts broader review).

### Good example
```js
srv.before('CREATE', 'Orders', async req => {
  const stock = await SELECT.one.from(Books, req.data.book_ID, b => b.stock);
  // context (tx, tenant, user) propagated automatically
});
```

### Bad example
```js
srv.before('CREATE', 'Orders', async req => {
  const tx = cds.tx(req);                       // legacy boilerplate
  const stock = await tx.run(SELECT.one.from(Books, req.data.book_ID));
});
```

### Exception guidance
None needed — the replacement is strictly simpler. Existing occurrences may be cleaned opportunistically rather than in a dedicated refactoring.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-tx ("still works but is not required nor recommended anymore"; continuation-local context propagation)

---

## CAP-TXN-004 — Control CAP Java transactions via the ChangeSet context or Spring transaction management

| Field | Value |
|---|---|
| **Rule ID** | CAP-TXN-004 |
| **Title** | Control CAP Java transactions via the ChangeSet context or Spring transaction management |
| **Category** | Transactions |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | CAP Java code needing explicit transactional control (cancellation, custom boundaries, background work) |
| **Runtime** | Java (Node.js counterpart: CAP-TXN-002) |
| **CAP version** | All currently supported versions; note `cds.persistence.changeSet.enforceTransactional` defaults to `false` (plain reads run without starting a transaction — a deliberate throughput optimization) |
| **Status** | Active |
| **Related rules** | CAP-TXN-001, CAP-TXN-005, CAP-MT-006 (RequestContextRunner for tenant context) |
| **Last verified** | 2026-08-11 |

### Rule statement
Explicit transactional control in CAP Java MUST use the documented integrated mechanisms: the ChangeSet Context API (`markForCancel()` to roll back without an exception; `ChangeSetListener.beforeClose()` to veto a commit; `requestContext()`/changeset nesting for custom boundaries) or Spring's transaction management (`@Transactional`, `TransactionTemplate`) — SAP: "CAP Java completely integrates with Spring's transaction management." Raw JDBC transaction handling (manual `Connection` commit/rollback) on the CAP datasource MUST NOT be used. The `enforceTransactional=false` default (reads without transactions) SHOULD NOT be changed without a recorded reason.

### Rationale
Both documented mechanisms keep CAP's changeset semantics, event timing, and connection handling intact; raw JDBC transaction control operates beneath the framework, breaking the coordination between changeset listeners, queued events (CAP-EVT-002), and the actual commit. Flipping `enforceTransactional` silently changes read-path resource behavior (every read starts holding a transaction/connection). **Medium:** scoped to explicit-control sites; the integrated APIs make correct behavior easy.

### Evidence expected in code
`ChangeSetContext`/`@Transactional`/`TransactionTemplate` at explicit-control sites; no manual `Connection`-level transaction code against the CAP datasource; default `enforceTransactional` untouched or its change recorded.

### Detection guidance
1. Search `srv/src/main/java/**` for `ChangeSetContext`, `markForCancel`, `ChangeSetListener`, `@Transactional`, `TransactionTemplate` — these are the compliant vocabulary; verify each use matches intent (veto vs cancel vs new boundary).
2. Search for raw JDBC transaction handling: `getConnection()` + `setAutoCommit(false)`/`commit()`/`rollback()` on the CAP-managed datasource → FAIL with file:line (documented native-SQL reads via `JdbcTemplate` per CAP-DB-004 are fine — the FAIL is *transaction* control, not query execution).
3. Check `application*.yaml` for `cds.persistence.changeSet.enforceTransactional: true` → recorded reason present → PASS with note; absent → FAIL (undocumented default change).
4. Cross-check background-work sites also establish tenant context (CAP-MT-006).
5. Report per site.

### Good example
```java
// veto the commit across the whole changeset if a final check fails
context.getChangeSetContext().register(new ChangeSetListener() {
  @Override public void beforeClose() {
    if (!consistencyCheck.ok()) throw new ServiceException(ErrorStatuses.CONFLICT, "inconsistent");
  }
});
```

### Bad example
```java
// raw JDBC transaction control under CAP's feet — changeset listeners
// and queued events no longer align with the actual commit
Connection c = dataSource.getConnection();
c.setAutoCommit(false);
stmt.executeUpdate("UPDATE ...");
c.commit();
```

### Exception guidance
Non-CAP datasources (separate technical databases) are outside CAP's transaction scope and may be managed with Spring's standard means — documented per CAP-ARCH-007 if they hold business data. No exception for the CAP datasource.

### SAP reference
- https://cap.cloud.sap/docs/java/event-handlers/changeset-contexts (ChangeSet API; `markForCancel`; `beforeClose` veto; Spring integration; `enforceTransactional` default)

---

## CAP-TXN-005 — Never assume distributed atomicity across transactions

| Field | Value |
|---|---|
| **Rule ID** | CAP-TXN-005 |
| **Title** | Never assume distributed atomicity across transactions |
| **Category** | Transactions |
| **Severity** | High |
| **Authority** | SAP-REQ (Node.js: documented limitation, verbatim; Java: the changeset documentation states boundaries are delegated to transaction managers and documents **no** cross-member atomicity guarantee — none may be assumed) |
| **Applicability** | All flows spanning multiple transactions: service-to-service calls with own transactions, multiple changesets (e.g., OData batch), remote calls, mixed local+remote work |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-EVT-002 (transactional queue = the documented reliable-side-effect mechanism), CAP-EVT-003 (idempotency), CAP-TXN-001; async-work compensation patterns are ORG gap G-33 |
| **Last verified** | 2026-08-11 |

### Rule statement
Code MUST NOT assume atomic commit across transaction boundaries. Node.js documents this explicitly for nested transactions: they are "*synchronized* with respect to a final commit / rollback, but *not as a distributed atomic transaction*. This means, it still can happen, that the commit of one nested transaction succeeds, while the other fails." CAP Java documents no cross-changeset or cross-resource atomicity guarantee. Consequently: (1) flows spanning transactions MUST handle partial success deliberately (compensation, idempotent retry, or reordering so the critical write commits last); (2) reliable coupled side effects (messages, audit, remote notifications) MUST ride the transactional event queue (CAP-EVT-002) instead of assuming they commit with the business data; (3) remote calls inside a local transaction MUST NOT be treated as rolled back when the local transaction fails.

### Rationale
This is the highest-risk misconception in the batch's scope: developers assume "it's all one request, so it's all one transaction" across service boundaries and remote calls. The documented reality is partial-commit windows — money moved in one system, order lost in the other. SAP's documented answer for coupled side effects is the queue ("no phantom events, no lost events"), not distributed transactions. **High:** violations produce cross-system inconsistency — a genuine integrity defect — but require multi-transaction flows to manifest (not every request), and CAP's defaults (single managed transaction per request, queued messaging) make the safe path the easy path.

### Implementation guidance
- Prefer designs where one local transaction owns the business write and everything else is a queued event (CAP-EVT-002) consumed idempotently (CAP-EVT-003).
- Where a flow genuinely spans transactions, write the partial-failure story down (what compensates what) — G-33 governs the org-level pattern standard.
- Never call a remote system mid-transaction and assume its effect vanishes on local rollback.

### Evidence expected in code
Multi-transaction flows with visible partial-failure handling (compensation handlers, idempotent design, queue usage); no remote state changes fired-and-forgotten inside local transactions.

### Detection guidance
1. Identify multi-transaction flows: `cds.tx()` calls creating separate roots, remote service `.send()`/emit-unqueued calls inside handlers, cross-deployment-unit synchronous call chains (cross-ref CAP-ARCH-006 observations).
2. For each, determine the partial-failure behavior: what happens if transaction A commits and B fails? Look for compensation logic, idempotent retry, or queue mediation.
3. No deliberate handling anywhere (and no documented acceptance) → FAIL with the flow's file:line entry points.
4. Remote *writes* inside local transactions without queue mediation or compensation → FAIL (cite the Node.js quote / absence of Java guarantee).
5. Check the report of CAP-EVT-002 for services that disabled queueing — those sites re-enter this rule's scope.
6. NOT ASSESSABLE where flow behavior needs runtime tracing — name the missing evidence (integration test of the failure path).

### Good example
```js
srv.on('submitOrder', async req => {
  await INSERT.into(Orders).entries(order);        // one local transaction…
  await srv.emit('OrderSubmitted', { order });     // …queued event delivers the
});                                                 // side effects after commit
```

### Bad example
```js
srv.on('submitOrder', async req => {
  await INSERT.into(Orders).entries(order);
  await billing.send('charge', { amount });   // remote charge inside local tx:
  await riskyFinalCheck(req);                  // if this throws, the order rolls
});                                            // back — the charge does NOT
```

### Exception guidance
None on the assumption itself. Flows with formally accepted partial-failure risk (low-stakes, reconciled nightly) document that acceptance (ADR/exception record) — the review then verifies the record, not compensation code.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-tx (nested transactions synchronized, "not as a distributed atomic transaction")
- https://cap.cloud.sap/docs/java/event-handlers/changeset-contexts (boundaries delegated to transaction managers; no atomicity guarantee documented)
- https://cap.cloud.sap/docs/guides/events/event-queues (queued work shares the business transaction)

---

## CAP-TXN-006 — Await every asynchronous operation in transactional contexts

| Field | Value |
|---|---|
| **Rule ID** | CAP-TXN-006 |
| **Title** | Await every asynchronous operation in transactional contexts |
| **Category** | Transactions |
| **Severity** | High |
| **Authority** | SAP-REQ |
| **Applicability** | All Node.js handler and `cds.spawn` code |
| **Runtime** | Node.js (Java's threading model has no direct counterpart; thread hand-off rules live in CAP-MT-006's context guidance) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-TXN-001, CAP-TXN-002, CAP-EVT-001 (emits are the most common victim); absorbs candidate CAP-LOGIC #5 |
| **Last verified** | 2026-08-11 |

### Rule statement
Inside handlers and `cds.spawn` callbacks, every asynchronous operation MUST be awaited — including every `srv.emit(...)`. SAP is explicit on both: emitters must "always [be called] with `await`" because "Not handling them will likely lead to invalid transaction states and deadlocks"; and in `cds.spawn`, "all asynchronous operations inside the callback function must be awaited. Otherwise, transaction handling does not work properly." Fire-and-forget promises in transactional contexts are prohibited; deliberate background work uses `cds.spawn` (itself fully awaited inside).

### Rationale
Un-awaited operations escape the transaction lifecycle: the framework finalizes (commits/rolls back and releases the connection) while the stray operation still runs against a finished transaction — producing invalid transaction states, deadlocks, and heisenbugs that appear under load. This is the classic async-CAP defect, invisible in happy-path tests. **High:** documented failure mode is transaction-integrity corruption; the violation is a one-keyword omission that reviews catch cheaply.

### Implementation guidance
- Treat every `srv.emit`, `srv.run`, `INSERT/UPDATE/DELETE/SELECT`, and remote `.send` as await-mandatory inside handlers.
- Parallelism inside a handler uses `await Promise.all([...])` — parallel *awaited*, never detached (mind CAP-TXN-002's SQLite parallel-transaction caveat in dev).
- Background work is `cds.spawn(async () => { …all awaited… })`, not a naked async call.
- Enable a lint rule (`no-floating-promises`) — it mechanically enforces this rule.

### Evidence expected in code
No floating promises in `srv/**`; `await srv.emit` throughout; lint configuration catching violations.

### Detection guidance
1. Search handler code for un-awaited calls: `srv.emit(` / `this.emit(` / `.send(` / `cds.run(`/`INSERT.`/`UPDATE.`/`DELETE.` expressions used as statements without `await`/`.then`/return → each → FAIL with file:line.
2. Check `cds.spawn` callbacks: async operations inside without `await` → FAIL.
3. Check for detached background patterns (`someAsync().catch(console.error)` inside handlers, `setTimeout` with DB access — the latter also CAP-MT-006) → FAIL here.
4. Check lint setup: `@typescript-eslint/no-floating-promises` or equivalent active → note as preventive control (absence is an observation, not FAIL).
5. Report per site.

### Good example
```js
srv.after('CREATE', 'Orders', async (order, req) => {
  await srv.emit('OrderCreated', { ID: order.ID });   // awaited — rides the
});                                                    // transaction correctly
```

### Bad example
```js
srv.after('CREATE', 'Orders', (order, req) => {
  srv.emit('OrderCreated', { ID: order.ID });   // floating promise: emits race
});                                              // transaction finalization →
                                                 // invalid tx states, deadlocks
```

### Exception guidance
None. Deliberately detached work is expressed through `cds.spawn` (with awaited internals) or the queue — never through floating promises.

### SAP reference
- https://cap.cloud.sap/docs/node.js/core-services ("always call [emitters] with `await`"; "invalid transaction states and deadlocks")
- https://cap.cloud.sap/docs/node.js/cds-tx (`cds.spawn`: "all asynchronous operations inside the callback function must be awaited")
