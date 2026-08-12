# CAP-EVT — Events & messaging

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-30, G-31, G-32, G-33 in [research-gaps.md](../../references/research-gaps.md).

**Rules:** 7 active (2 Critical, 2 High, 2 Medium, 1 Low). All SAP references verified against official CAP documentation on **2026-08-11**. Delivery-guarantee statements below are deliberately limited to what SAP documents — notably: **exactly-once is not guaranteed** ("handlers must still be idempotent"), and the transactional queue is the **default**, not an add-on.

**Applicability precondition:** rules 001–005 apply to projects emitting or consuming asynchronous events/messages (or using queued processing, incl. audit logging via the queue); 006–007 to projects using a message broker. Projects with no eventing are NOT APPLICABLE throughout — state the evidence.

Scope boundaries: tenant context in queued/background processing is [CAP-MT-006](multitenancy.md); no-distributed-atomicity design is [CAP-TXN-005](transactions.md); S/4- and broker-*integration* specifics are [CAP-INT](integration.md).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-EVT-001 | Declare and emit events protocol-agnostically | Medium | SAP-REC | Both |
| CAP-EVT-002 | Keep the transactional event queue on for production messaging and audit | Critical | SAP-REC | Both |
| CAP-EVT-003 | Queued and event handlers are idempotent | Critical | SAP-REQ | Both |
| CAP-EVT-004 | Queued processing runs privileged — carry claims explicitly, keep secrets out of headers | High | SAP-REQ | Both |
| CAP-EVT-005 | Operate the dead-letter queue deliberately | High | SAP-REC | Both |
| CAP-EVT-006 | Use SAP Cloud Application Event Hub as the default broker for new BTP apps | Medium | SAP-REC | Both |
| CAP-EVT-007 | Local development uses local messaging | Low | SAP-REC | Both |

---

## CAP-EVT-001 — Declare and emit events protocol-agnostically

| Field | Value |
|---|---|
| **Rule ID** | CAP-EVT-001 |
| **Title** | Declare and emit events protocol-agnostically |
| **Category** | Events & messaging |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | All event emission and consumption |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ARCH-002/-005 (hand-rolled broker clients are framework bypasses — escalate there), CAP-TXN-006 (`await` every emit), CAP-EVT-006; event naming/versioning schemes are ORG gap G-30 |
| **Last verified** | 2026-08-11 |

### Rule statement
Events MUST be declared in the CDS model (`event <Name> : { … }`), emitted via `srv.emit(...)`, and consumed via `srv.on(...)` (or the Java equivalents) — leaving wire formats, broker transport, and tenant handling to the framework: SAP: "Simply use `srv.emit()` to emit events, and let the CAP framework care for wire protocols like CloudEvents, transports via message brokers, multitenancy handling, and so forth." Application code MUST NOT hand-roll broker clients, topic-string plumbing, or CloudEvents envelopes where the framework mechanism covers the requirement.

### Rationale
Modeled events are typed API contract (visible in the model, AsyncAPI-exportable), broker-portable (local messaging in dev, Event Hub in production — CAP-EVT-007/-006), and tenant-correct by framework handling. Hand-rolled broker code loses all three and bypasses the transactional queue (CAP-EVT-002). **Medium:** the direct violation is coupling/duplication; where it bypasses queueing or tenant handling it escalates through CAP-EVT-002/CAP-MT-003, and as a framework bypass through CAP-ARCH-002.

### Evidence expected in code
`event` declarations in `srv/**/*.cds`; `srv.emit`/`srv.on` (or Java `EventContext`-based emit/handlers); no direct AMQP/HTTP broker client code in application logic.

### Detection guidance
1. Search for direct broker usage in `srv/**`: AMQP/Kafka/HTTP-webhook client libraries, hand-built CloudEvents JSON, raw topic-string publishing outside CAP messaging config → each → FAIL with file:line (escalate to CAP-ARCH-002 if it forms a layer).
2. Verify emitted events are declared in the model (`event` definitions) rather than magic strings only → undeclared event contracts → FAIL (Medium) per event.
3. Verify consumption uses `srv.on('<Event>')` against connected services (agnostic form) rather than low-level topic subscriptions where the agnostic form is documented (e.g., S/4 events).
4. Cross-check every emit site is awaited (CAP-TXN-006).
5. NOT APPLICABLE if the project has no eventing.

### Good example
```cds
service ReviewsService {
  event Reviewed : { subject : String; rating : Decimal; }
}
```
```js
await this.emit('Reviewed', { subject, rating });          // emitter
const reviews = await cds.connect.to('ReviewsService');
reviews.on('Reviewed', msg => updateRating(msg.data));     // consumer
```

### Bad example
```js
// hand-rolled broker client + hand-built envelope in application logic:
// no model contract, no queue, no tenant handling, broker-locked
const amqp = require('amqplib');
const ch = await (await amqp.connect(BROKER_URL)).createChannel();
ch.publish('reviews', 'reviewed.v1', Buffer.from(JSON.stringify({ specversion: '1.0', data })));
```

### Exception guidance
Brokers/protocols genuinely outside CAP's messaging support may need a dedicated integration client — isolated, documented per CAP-ARCH-007, and still queue-mediated where reliability matters. Event *naming/versioning* conventions are ORG policy (G-30), not this rule.

### SAP reference
- https://cap.cloud.sap/docs/guides/events/core-concepts (declare in CDS; emit/consume; framework handles CloudEvents, transports, multitenancy)

---

## CAP-EVT-002 — Keep the transactional event queue on for production messaging and audit

| Field | Value |
|---|---|
| **Rule ID** | CAP-EVT-002 |
| **Title** | Keep the transactional event queue on for production messaging and audit |
| **Category** | Events & messaging |
| **Severity** | Critical |
| **Authority** | SAP-REC (documented default with documented rationale; the immediate/unqueued variant is a documented API — hence not SAP-REQ) |
| **Applicability** | Production configurations of projects emitting messages or audit-log events |
| **Runtime** | Both |
| **CAP version** | ⏱ Queue enabled by default in current versions. **Upgrade hazard (documented):** rolling upgrade `@sap/cds` 8 → 10 can double-process queued messages — plan downtime, drain the queue, or upgrade through v9 |
| **Status** | Active |
| **Related rules** | CAP-TXN-005 (the queue is the documented answer to no-distributed-atomicity), CAP-EVT-003, CAP-EVT-005 |
| **Last verified** | 2026-08-11 |

### Rule statement
Production messaging and audit-log delivery MUST go through the persistent transactional event queue — the documented default ("Event queues are enabled by default — there's nothing to install or activate"), under which "Queued work is written in the same transaction as your business data → no phantom events, no lost events." Disabling queueing for a service (immediate dispatch via `cds.unqueued(...)`, per-service `outboxed: false`-style configuration, or in-memory-only queue configs in production) requires a documented justification explicitly accepting the documented consequences: events sent although the transaction rolled back (phantom events) and events lost on crash before dispatch (lost events).

### Rationale
The queue is what makes emitted events and audit entries consistent with business data: "**No phantom events**: if the transaction rolls back, no message is sent. **No lost events**: if the transaction commits, the queued work is persisted and processed eventually." Messaging services and the audit-log service use it by default. **Critical justification:** silently disabling it in production means integration partners receive events for rolled-back business transactions and audit-relevant operations can vanish on crashes — cross-system data inconsistency and lost audit evidence at scale, discovered only during incident forensics. This is production-integrity failure in SAP's own documented terms (phantom/lost events).

### Implementation guidance
- Do nothing — the default is compliance. Treat any `unqueued`/immediate-dispatch usage as an exception needing a recorded reason (legitimate example: a synchronous probe where delivery-or-fail-now is the point).
- During major upgrades crossing cds 8 → 10, follow the documented queue-drain guidance (version field above).

### Evidence expected in code
No production configuration disabling the queue for messaging/audit services; `cds.unqueued`/immediate-dispatch call sites carrying documented justifications.

### Detection guidance
1. Search configuration (package.json, `.cdsrc*`, `application*.yaml`) for queue/outbox overrides on messaging or audit-log services: `outboxed: false`, in-memory queue kinds in production profiles, disabled-queue flags → each in a production-reaching profile → FAIL unless a documented justification exists.
2. Search code for `cds.unqueued(` (Node) / `OutboxService.unboxed(`-style immediate variants (Java) → per site, locate the justification comment/ADR → absent → FAIL.
3. Verify audit logging (if present per CAP-PRIV scope) delivers through the queue (default) — overridden → same treatment.
4. For upgrade reviews (cds 8 → 10 in progress): check the runbook for the documented drain/step-through-9 mitigation → absent → FAIL (upgrade-hazard element).
5. NOT APPLICABLE if the project emits no messages and uses no queued services.

### Good example
```js
// default behavior — nothing configured: emit is queued, committed with
// the business data, dispatched after commit
await srv.emit('OrderCreated', { ID });
```

### Bad example
```jsonc
// production messaging switched to immediate/in-memory dispatch —
// phantom events on rollback, lost events on crash, silently
{ "cds": { "requires": { "messaging": { "kind": "enterprise-messaging", "outboxed": false } } } }
```

### Exception guidance
Deliberate immediate dispatch where the caller must observe delivery failure synchronously (health probes, interactive "test connection" actions) — documented at the site. Dev/test profiles may use in-memory queues freely; the rule binds production-reaching configuration.

### SAP reference
- https://cap.cloud.sap/docs/guides/events/event-queues (enabled by default; same-transaction semantics; "no phantom events, no lost events"; messaging & audit-log services queued by default; cds 8→10 double-processing hazard)

---

## CAP-EVT-003 — Queued and event handlers are idempotent

| Field | Value |
|---|---|
| **Rule ID** | CAP-EVT-003 |
| **Title** | Queued and event handlers are idempotent |
| **Category** | Events & messaging |
| **Severity** | Critical |
| **Authority** | SAP-REQ ("handlers must still be idempotent") |
| **Applicability** | All consumers of queued work: event/message handlers, queued task processors |
| **Runtime** | Both |
| **CAP version** | All currently supported versions (semantics of the current transactional event queue) |
| **Status** | Active |
| **Related rules** | CAP-EVT-002, CAP-EVT-005, CAP-MT-004 (the tenant-lifecycle sibling of this rule), CAP-TXN-005 |
| **Last verified** | 2026-08-11 |

### Rule statement
Every handler processing queued or asynchronous events MUST be idempotent — safe under redelivery and duplicate execution. SAP documents the guarantee boundary precisely: "CAP avoids duplicate execution under normal operation, but handlers **must still be idempotent** to tolerate rare crash windows or external side effects." Exactly-once MUST NOT be assumed anywhere. Idempotency is achieved by design: natural idempotence (absolute writes keyed by the event's subject), duplicate detection (processed-event bookkeeping keyed by event ID), or check-then-act against current state — chosen per handler and visible in the code.

### Rationale
Redelivery is not hypothetical: crash windows between processing and acknowledgment, retries after partial failures (CAP-EVT-005), and the documented cds 8→10 double-processing upgrade hazard (CAP-EVT-002) all produce duplicates. A non-idempotent handler then executes business effects twice — double postings, double emails, double stock movements. **Critical justification:** duplicate execution of business effects is silent data corruption of exactly the kind SAP's wording anticipates; unlike the High-rated tenant-lifecycle sibling (CAP-MT-004, provisioning artifacts), the blast radius here is business data and external side effects. This mirrors the documented at-least-once reality — never a manufactured guarantee.

### Implementation guidance
- Prefer *naturally idempotent* handlers: `UPSERT`/absolute state writes keyed by the event subject ("set rating of book X to 4.2") over relative mutations ("add 0.3 to rating").
- Where effects are inherently one-shot (send email, call external API), record-and-skip: persist the processed event ID in the same transaction as the effect's bookkeeping, skip on re-encounter.
- Treat external side effects specially — they escape the local transaction (CAP-TXN-005); their idempotency lives with the receiver (idempotency keys) or the bookkeeping.

### Evidence expected in code
Handlers using absolute writes / duplicate-detection bookkeeping / documented check-then-act; tests that deliver the same event twice and assert single effect.

### Detection guidance
1. Enumerate async consumers: `srv.on('<Event>')` on connected/messaging services, queued task processors (Node); `@On` handlers for events / outboxed service consumers (Java).
2. Classify each handler's effects: relative mutations (`+=`-style updates, `INSERT` without existence semantics, counters), external calls without idempotency keys → non-idempotent candidates.
3. For each candidate, look for the idempotency mechanism (absolute write semantics, event-ID bookkeeping, existence checks) → none → FAIL with file:line and the mechanism recommendation.
4. Look for a duplicate-delivery test (same event twice → one effect) per critical handler → absent → report as missing evidence element.
5. NOT APPLICABLE if no async consumers exist.

### Good example
```js
messaging.on('ReviewAdded', async msg => {
  const { subject, avgRating } = msg.data;
  // absolute write keyed by subject — safe under any redelivery
  await UPSERT.into(Ratings).entries({ book_ID: subject, rating: avgRating });
});
```

### Bad example
```js
messaging.on('ReviewAdded', async msg => {
  // relative mutation — every duplicate delivery shifts the sum again
  await UPDATE(Books, msg.data.subject).set('ratingSum = ratingSum + ' + msg.data.rating);
});
```

### Exception guidance
None on the requirement itself. Handlers whose duplicate execution is provably harmless (pure cache invalidation, idempotent-by-math effects) satisfy the rule inherently — state why in a comment so reviews don't re-analyze.

### SAP reference
- https://cap.cloud.sap/docs/guides/events/event-queues ("handlers must still be idempotent to tolerate rare crash windows or external side effects")

---

## CAP-EVT-004 — Queued processing runs privileged — carry claims explicitly, keep secrets out of headers

| Field | Value |
|---|---|
| **Rule ID** | CAP-EVT-004 |
| **Title** | Queued processing runs privileged — carry claims explicitly, keep secrets out of headers |
| **Category** | Events & messaging |
| **Severity** | High |
| **Authority** | SAP-REQ (documented processing semantics and explicit header warning) |
| **Applicability** | All queued/outboxed processing and outbound messaging |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-MT-006 (tenant context restored; technical users for async), CAP-SEC-016 (sensitive data hygiene), CAP-SEC-009 |
| **Last verified** | 2026-08-11 |

### Rule statement
Handlers of queued work MUST NOT rely on the original caller's roles, attributes, or tokens: SAP documents that only "the **user ID** is stored with the queued message… **User roles, attributes, and tokens** are *not* stored. Asynchronous processing always runs in privileged mode", with "no principal propagation… across the queue boundary, by design". Where the original caller's identity matters to processing, the relevant claims MUST be captured explicitly at queue time (payload preferred) and evaluated in the handler. Outbound queued calls forward stored headers to their targets — per SAP: "Do not carry sensitive data — authentication tokens, personal data, secrets — in headers on outbound calls."

### Rationale
Two distinct failures hide here. (1) *Correctness/authorization:* a handler checking `user.is('admin')` in queued processing checks a privileged technical context, not the original caller — authorization decisions silently evaluate against the wrong identity. (2) *Confidentiality:* headers persist in the queue table and are forwarded to external targets — tokens/PII placed there leak beyond their intended scope and lifetime. **High:** both failures are real and security-adjacent, but require specific mistakes (role checks in async handlers; sensitive data deliberately placed in headers) rather than being a default-open condition.

### Implementation guidance
- Decide authorization *before* queueing, while the real caller context exists; queue the decision or the needed claims (payload), not the assumption.
- Handlers treat their privileged mode as a responsibility: re-derive tenant-scoped access from the restored tenant (CAP-MT-006), never from assumed roles.
- Keep message headers for routing metadata only.

### Evidence expected in code
Queue-time capture of needed claims in payloads; no role/attribute checks against the processing context inside async handlers; headers free of tokens/PII/secrets.

### Detection guidance
1. Enumerate async/queued handlers (as in CAP-EVT-003 step 1).
2. Search their bodies for caller-identity reliance: `req.user.is(`, `user.attr.`, `UserInfo.hasRole(` etc. evaluated on the processing context → each → FAIL (identity is the privileged technical user).
3. Inspect emit/queue sites: headers populated with `authorization`, tokens, personal data, secrets → FAIL with file:line (documented prohibition).
4. Where handler logic legitimately needs caller claims: verify they are captured at queue time into the payload and read from there → absent → FAIL.
5. Cross-check tenant handling (restored automatically) is not overridden with hardcoded tenants (CAP-MT-003/-006).

### Good example
```js
// queue time: capture the decision inputs explicitly
await srv.emit('RefundRequested', {
  order_ID, amount,
  requestedBy: req.user.id, requestorWasManager: req.user.is('manager')
});
// processing time: use the captured claim, not the (privileged) context
messaging.on('RefundRequested', async msg => {
  if (!msg.data.requestorWasManager && msg.data.amount > LIMIT) return escalate(msg.data);
  await processRefund(msg.data);
});
```

### Bad example
```js
messaging.on('RefundRequested', async (msg) => {
  // processing runs privileged — this check is meaningless (and passes)
  if (cds.context.user.is('manager')) await processRefund(msg.data);
});
// and at emit time:
await srv.emit('RefundRequested', data, { authorization: req.headers.authorization }); // token persisted & forwarded
```

### Exception guidance
Technical headers required by target protocols (correlation IDs, content types) are fine. No exception for tokens/PII/secrets in headers, and none for role checks against the privileged processing context.

### SAP reference
- https://cap.cloud.sap/docs/guides/events/event-queues (user ID only; privileged mode; no principal propagation by design; claims via payload/headers at queue time; no sensitive data in outbound headers)

---

## CAP-EVT-005 — Operate the dead-letter queue deliberately

| Field | Value |
|---|---|
| **Rule ID** | CAP-EVT-005 |
| **Title** | Operate the dead-letter queue deliberately |
| **Category** | Events & messaging |
| **Severity** | High |
| **Authority** | SAP-REC (documented behavior + documented operational patterns; thresholds/ownership are ORG — gap G-32) |
| **Applicability** | Production projects using queued processing |
| **Runtime** | Both |
| **CAP version** | Current queue semantics (default `maxAttempts: 10`, exponential backoff) |
| **Status** | Active |
| **Related rules** | CAP-EVT-002, CAP-EVT-003, CAP-SEC-009 (dead-letter admin actions need protected services); CAP-LOG-004 (telemetry mechanism); alerting thresholds remain ORG (G-32/G-37) |
| **Last verified** | 2026-08-11 |

### Rule statement
Projects using queued processing MUST have a deliberate dead-letter operation: SAP documents that after the configurable maximum attempts (default 10, exponential backoff) — or immediately for errors "identified as *unrecoverable*" — messages move to the dead-letter queue, "remain in the `cds.outbox.Messages` database table with their error information intact… and require manual intervention." Concretely required: (1) queue/dead-letter monitoring wired (SAP: "Both stacks export queue metrics through OpenTelemetry"); (2) an operational path to inspect, revive, or delete dead letters (SAP documents a CDS service pattern with bound *revive*/*delete* actions — such a service MUST be access-restricted per CAP-SEC-001/-009); (3) handlers marking semantic (non-retryable) errors as unrecoverable — a **developer-implemented** pattern per SAP's examples (Node: `e.unrecoverable = true`; Java: catch and `context.setCompleted()`), not automatic framework behavior. Alert thresholds, ownership, and SLAs remain ORG policy (G-32).

### Rationale
Without a dead-letter process, permanently failing messages accumulate silently — each one a business event that never happened (a lost order notification, an unsent audit record), discovered weeks later. Retrying semantic errors (validation failures) ten times with backoff merely delays the same dead letter while burning processing windows. **High:** unmonitored dead letters are lost business events — a production-reliability failure — but the queue itself preserves the data (recoverable), which distinguishes this from Critical.

### Implementation guidance
- In handlers, classify errors: transient (let retry) vs semantic (mark unrecoverable immediately — SAP's example pattern treats 4xx-class semantic errors this way).
- Expose the documented dead-letter admin service pattern, filter `attempts >= maxAttempts` programmatically (maxAttempts is configurable), and restrict it to an operations role.
- Wire the OpenTelemetry queue metrics into the project's monitoring (mechanism per CAP-LOG-004; alerting policy remains ORG — G-32/G-37).

### Evidence expected in code
Error-classification in queued handlers; a restricted dead-letter admin service (or documented equivalent operational procedure); queue metrics in the monitoring setup; ORG thresholds referenced (G-32) or flagged open.

### Detection guidance
1. Confirm queued processing exists (else NOT APPLICABLE).
2. Search queued handlers for error classification (`unrecoverable`, `setCompleted()` on semantic failures) → none anywhere, while handlers can fail semantically → FAIL element (everything retries 10× then dead-letters).
3. Locate the dead-letter operational path: admin service with revive/delete actions, or documented runbook procedure against `cds.outbox.Messages` → none → FAIL.
4. If an admin service exists: verify `@requires` restriction (cross-check CAP-SEC-001) → unrestricted → FAIL (report also under CAP-SEC-001).
5. Check monitoring config for queue metrics (OTel exporter wired; dashboard/alert references) → none → FAIL element; thresholds undefined → observation referencing G-32.
6. Report per element with locations.

### Good example
```js
messaging.on('OrderNotification', async msg => {
  try { await notify(msg.data); }
  catch (e) {
    if (e.code >= 400 && e.code < 500) e.unrecoverable = true;  // semantic — don't retry
    throw e;
  }
});
```
```cds
@requires: 'QueueOperator'                       // restricted admin access
service DeadLetterService { /* revive/delete actions per SAP's documented pattern */ }
```

### Bad example
```text
Queued processing in production; no error classification, no dead-letter
service, no queue metrics. Failing notifications retry 10× over hours,
then sit invisible in cds.outbox.Messages — customers never notified,
nobody alerted.
```

### Exception guidance
Low-stakes queues (best-effort cache refresh) may document delete-on-dead-letter as the policy — the *deliberate decision* is the requirement. The unrecoverable-marking element is NOT APPLICABLE for handlers that genuinely cannot fail semantically.

### SAP reference
- https://cap.cloud.sap/docs/guides/events/event-queues (retries/backoff, default max 10; dead letters "require manual intervention"; unrecoverable-error pattern; revive/delete service pattern; OpenTelemetry queue metrics)

---

## CAP-EVT-006 — Use SAP Cloud Application Event Hub as the default broker for new BTP apps

| Field | Value |
|---|---|
| **Rule ID** | CAP-EVT-006 |
| **Title** | Use SAP Cloud Application Event Hub as the default broker for new BTP apps |
| **Category** | Events & messaging |
| **Severity** | Medium |
| **Authority** | SAP-REC ("the new default offering for messaging in SAP BTP") |
| **Applicability** | New BTP projects selecting a message broker; existing Event Mesh installations out of scope |
| **Runtime** | Both (different plugins: Node `@cap-js/event-broker` / Java `cds-feature-event-hub`) |
| **CAP version** | ⏱ Version-/portfolio-sensitive: broker positioning changes with SAP's messaging portfolio — re-verify at each re-verification cycle. Requires a productive (paid) BTP account |
| **Status** | Active |
| **Related rules** | CAP-EVT-001 (agnostic code makes the broker swappable), CAP-EVT-007; broker selection matrix is ORG gap G-31 |
| **Last verified** | 2026-08-11 |

### Rule statement
New BTP applications selecting a message broker SHOULD use SAP Cloud Application Event Hub — SAP: "the new default offering for messaging in SAP BTP" — with the documented setup (Node: `@cap-js/event-broker`; Java: `cds-feature-event-hub`; IAS/X509 binding requirements). Choosing SAP Event Mesh or a non-SAP broker for a new project requires a documented reason (existing landscape, feature constraints — ORG selection criteria per G-31). Application code stays broker-agnostic per CAP-EVT-001, so this is a configuration decision, not a code decision.

### Rationale
Aligning new projects with SAP's stated default keeps them on the maintained integration path (plugins, IAS-based security model, S/4 event sources). Because CAP-EVT-001 keeps code agnostic, the cost of following the default is configuration only. **Medium:** a strategic-alignment recommendation; a differently chosen broker is not a defect when reasoned, and Event Mesh remains documented for existing setups.

### Evidence expected in code
Production messaging kind `event-broker` (Node) / Event Hub feature (Java) with the documented binding setup; or an ADR naming the alternative-broker reason.

### Detection guidance
1. Read production messaging configuration (package.json `cds.requires.messaging` `[production]`; Java features/bindings).
2. Event Hub configured → PASS.
3. Event Mesh/non-SAP broker on a **new** project → locate the documented reason → present → PASS with observation; absent → FAIL (Medium).
4. Existing Event Mesh installations → NOT APPLICABLE (note G-31 for future selection policy).
5. Verify local/dev profiles don't point at the production broker (cross-check CAP-EVT-007).

### Good example
```jsonc
{ "cds": { "requires": {
    "messaging": { "kind": "local-messaging",
      "[production]": { "kind": "event-broker" } }   // Event Hub in production
} } }
```

### Bad example
```text
Greenfield 2026 BTP app wired to SAP Event Mesh with no recorded reason —
against SAP's stated default, with no selection rationale for reviewers.
```

### Exception guidance
The documented reasons *are* the mechanism: landscape constraints, required features, existing enterprise broker strategy — recorded per CAP-ARCH-007/G-31. Trial/dev-only projects without production messaging are NOT APPLICABLE.

### SAP reference
- https://cap.cloud.sap/docs/guides/events/event-hub ("the new default offering for messaging in SAP BTP"; setup and binding requirements)

---

## CAP-EVT-007 — Local development uses local messaging

| Field | Value |
|---|---|
| **Rule ID** | CAP-EVT-007 |
| **Title** | Local development uses local messaging |
| **Category** | Events & messaging |
| **Severity** | Low |
| **Authority** | SAP-REC ("we recommend to use a messaging service of kind `local-messaging` for local testing") |
| **Applicability** | Local development/test profiles of projects using messaging |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-EVT-006, CAP-EVT-001, CAP-ARCH-005 (agnosticism enables this), CAP-SEC-017 (no production broker credentials locally) |
| **Last verified** | 2026-08-11 |

### Rule statement
Local development and default test profiles SHOULD use `local-messaging` (or file-based messaging for multi-process local setups) instead of binding cloud brokers: SAP documents that Event Hub "sends events via HTTP", so "you won't be able to receive events on your local machine unless you use a tunneling service", and recommends `local-messaging` for local testing. Cloud-broker access from developer machines is the exception (hybrid testing via `cds bind`), not the default.

### Rationale
Local messaging keeps the inner loop self-contained (CAP's documented "airplane mode" value), avoids developer-held production broker credentials (CAP-SEC-017), and avoids polluting shared brokers with dev traffic. The documented HTTP-delivery limitation means a cloud-broker-bound local setup silently receives nothing — a confusing non-error. **Low:** developer-experience and hygiene; nothing in production is affected.

### Evidence expected in code
Default/dev messaging kind `local-messaging` (or file-based), production broker only under `[production]`; hybrid access via `cds bind` where cloud-broker testing is genuinely needed.

### Detection guidance
1. Read messaging configuration per profile.
2. Default (non-production) profile binding a cloud broker (`event-broker`, `enterprise-messaging`) → FAIL (Low) with the config location; via documented hybrid setup (`cds bind`, no persisted credentials) → PASS with note.
3. Verify tests don't require broker connectivity to pass (CI runs without cloud messaging bindings).
4. NOT APPLICABLE without messaging.

### Good example
```jsonc
{ "cds": { "requires": { "messaging": {
    "kind": "local-messaging",
    "[production]": { "kind": "event-broker" }
} } } }
```

### Bad example
```jsonc
// default profile bound to the cloud broker: local emits go to the shared
// broker; local consumers receive nothing (HTTP delivery can't reach localhost)
{ "cds": { "requires": { "messaging": { "kind": "event-broker" } } } }
```

### Exception guidance
Deliberate hybrid tests against real brokers (webhook delivery verification via a tunneling service, contract tests in CI with bound credentials via `cds bind --exec`) are legitimate and documented — they complement, not replace, the local default.

### SAP reference
- https://cap.cloud.sap/docs/guides/events/event-hub (HTTP delivery; localhost limitation "unless you use a tunneling service"; `local-messaging` recommendation)
