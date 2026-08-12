# CAP-ERR — Error handling

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gap: G-02 (application error-code taxonomy — SAP documents mechanics, not a naming scheme).

**Rules:** 6 active (0 Critical, 3 High, 3 Medium). All SAP references verified against official CAP documentation on **2026-08-12**.

Scope boundaries: this category owns **error-response mechanics** — rendering, status mapping, sanitization, localization, message safety. Validation *timing and phase placement* is [CAP-LOGIC-002](business-logic.md); which inputs must be validated is [CAP-SEC-012](security.md); log content is [CAP-SEC-016](security.md)/[CAP-LOG](logging-observability.md); remote-error resilience is [CAP-INT-007](integration.md); queued-processing failures are [CAP-EVT-005](events-messaging.md). SAP documents no client/business/technical error taxonomy — none is invented here (org taxonomy = gap G-02).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-ERR-001 | Signal client errors through CAP's error APIs | Medium | SAP-REC | Both |
| CAP-ERR-002 | Keep production error sanitization intact | High | SAP-REQ | Node.js |
| CAP-ERR-003 | Fail loudly — never swallow errors | High | SAP-REC | Both |
| CAP-ERR-004 | Keep Node.js custom error handlers synchronous | Medium | SAP-REQ | Node.js |
| CAP-ERR-005 | Localize error messages via stable codes and targets | Medium | SAP-REC | Both |
| CAP-ERR-006 | Validate user input embedded in messages | High | SAP-REQ | Both |

---

## CAP-ERR-001 — Signal client errors through CAP's error APIs

| Field | Value |
|---|---|
| **Rule ID** | CAP-ERR-001 |
| **Title** | Signal client errors through CAP's error APIs |
| **Category** | Error handling |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented mechanisms; the behaviors — status mapping, collection, rollback — are documented facts) |
| **Applicability** | All custom handlers signaling request outcomes |
| **Runtime** | Both (mechanics differ per runtime — see statement) |
| **CAP version** | All currently supported versions; ⏱ CAP Java 5: `cds.errors.preferServiceException` default flipped to `true` — re-verify exception-vs-messages behavior at majors |
| **Status** | Active |
| **Related rules** | CAP-LOGIC-002 (phase/timing — not restated), CAP-ERR-005, CAP-SRV-004; absorbs candidates CAP-ERR #1 and #5 |
| **Last verified** | 2026-08-12 |

### Rule statement
Client-facing failures MUST be signaled through CAP's error APIs so the protocol adapters render correct responses: **Node.js** — `req.reject(status, code, …)` (constructs and throws, "sent back to the client in an error response") for immediate aborts, `req.error(…)` to collect multiple failures into `req.errors` for a combined response; **Java** — `ServiceException` with an `ErrorStatuses` mapping (without one "it defaults to an internal server error (HTTP status code 500)"), and the `Messages` API for collecting (error messages auto-trigger a `ServiceException` at the end of the `Before` phase via `throwIfError()`). Business rejections MUST NOT surface as raw generic exceptions (plain `throw new Error(...)` / unmapped `RuntimeException`), which reach clients as generic 500s instead of meaningful 4xx responses.

### Rationale
The error APIs are what connect a handler's decision to protocol-correct output: HTTP status, OData error format, i18n lookup (CAP-ERR-005), Fiori targets, and — in Java — the documented rollback semantics (any handler exception "causes any active transaction to be rolled back"). A validation failure thrown as a raw error becomes a 500: clients can't distinguish "your input is wrong" from "the server is broken", monitoring pages ops for user typos, and sanitization (CAP-ERR-002) hides the actual reason. **Medium:** API-quality and diagnosability; nothing is exposed or corrupted.

### Implementation guidance
- Choose by intent: definitely-fatal input → `reject`/throw `ServiceException` with a 4xx `ErrorStatuses`; validation with potentially several findings → collect (`req.error` / `Messages.error`) so consumers see everything at once.
- Give every business error a stable `code` (feeds CAP-ERR-005 and CAP-TEST-003); the code scheme itself is org policy (G-02).
- Let unexpected technical errors propagate (CAP-ERR-003) — they *should* be 500s.

### Evidence expected in code
Handlers using `req.error`/`req.reject` (Node) or `ServiceException`+`ErrorStatuses`/`Messages` (Java) for business failures; no raw generic throws for client-caused conditions.

### Detection guidance
1. Search Node handlers for `throw new Error(`/`Promise.reject(` signaling client-caused conditions (input problems, business-state conflicts) → FAIL per site (name the `req.reject`/`req.error` replacement).
2. Search Java handlers for unmapped throws on client-caused conditions (`RuntimeException`, `IllegalArgumentException`) and `ServiceException` without `ErrorStatuses` where a 4xx is meant (defaults to 500) → FAIL per site.
3. Verify multi-finding validation collects (Node `req.error`; Java `Messages`) rather than serially rejecting the first finding — observation unless combined reporting is a stated requirement (cross-ref CAP-LOGIC-002 step 5).
4. Confirm business errors carry stable `code` values (feeds CAP-ERR-005/CAP-TEST-003 checks).
5. Report per handler with file:line.

### Good example
```js
srv.before('CREATE', 'Orders', req => {
  if (!req.data.items?.length) req.error(400, 'ORDER_EMPTY', 'items');
  if (req.data.quantity > 1000) req.error(400, 'QUANTITY_LIMIT', 'quantity');
});  // both findings returned together, as 400 with stable codes
```

### Bad example
```java
@Before(event = CqnService.EVENT_CREATE, entity = Orders_.CDS_NAME)
void validate(List<Orders> orders) {
  if (orders.get(0).getItems().isEmpty())
    throw new RuntimeException("no items!");   // client sees generic 500;
}                                               // ops gets paged for a user typo
```

### Exception guidance
Genuine server-side faults (unreachable dependency after CAP-INT-007's handling, invariant violations) are correctly *not* client errors — let them propagate as 5xx (CAP-ERR-003). Framework-internal errors need no wrapping.

### SAP reference
- https://cap.cloud.sap/docs/node.js/events (`req.reject` throws; `req.error` collects into `req.errors`)
- https://cap.cloud.sap/docs/java/event-handlers/indicating-errors (`ServiceException`/`ErrorStatuses`, default 500; `Messages`, `throwIfError()`; rollback on handler exceptions)

---

## CAP-ERR-002 — Keep production error sanitization intact

| Field | Value |
|---|---|
| **Rule ID** | CAP-ERR-002 |
| **Title** | Keep production error sanitization intact |
| **Category** | Error handling |
| **Severity** | High |
| **Authority** | SAP-REQ ("In production, error responses should never disclose internal information that could be exploited by attackers") |
| **Applicability** | Node.js projects with production targets (see runtime note for Java) |
| **Runtime** | Node.js — Java note: CAP Java documents no security-worded sanitization switch — its documented default is that unexpected errors/broken error handlers collapse to "a generic internal server error with HTTP status 500 … not display any error details"; the reviewable Java duty is therefore not to *put* internal details into `ServiceException` messages (see detection step 4), since those are rendered to clients |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-016 (log-side twin), CAP-ERR-001, CAP-ERR-003; downgraded from candidate Critical |
| **Last verified** | 2026-08-12 |

### Rule statement
The production 5xx sanitization MUST stay intact: all 5xx errors are returned "with only the respective generic message" — code MUST NOT set `err.$sanitize = false` in any production-reaching path (SAP's own caution: "Use that option with care!"), and custom error handlers (`srv.on('error')`) MUST NOT copy internal details (stack traces, database errors, connection strings, tenant internals) into outgoing messages. In CAP Java, the equivalent duty: `ServiceException` messages are client-visible — internal diagnostic detail belongs in logs (CAP-SEC-016), never in the exception message.

### Rationale
SAP words this as a security requirement: production error responses "should never disclose internal information that could be exploited by attackers". Stack traces and DB errors leak schema names, file paths, library versions, and occasionally data — free reconnaissance. **High (downgraded from candidate Critical):** information *disclosure* that aids attack, not itself unauthorized access — consistent with the severity calibration of CAP-SEC-016.

### Evidence expected in code
No `$sanitize = false` anywhere production-reaching; error handlers that only augment safe fields; Java exception messages free of internal diagnostics (which live in log statements instead).

### Detection guidance
1. Search the codebase for `$sanitize` → any `= false` in production-reaching code → FAIL with file:line (dev-profile-guarded usage → observation with the guard shown).
2. Inspect `srv.on('error')` handlers: copying of `err.stack`, raw driver/database error text, or internal identifiers into `err.message`/outgoing fields → FAIL.
3. Node: check no middleware re-exposes error internals (custom Express error middleware serializing full errors) → FAIL.
4. Java: sample `ServiceException` construction sites — messages embedding stack traces, SQL text, or internal state → FAIL per site (detail belongs in the log call next to it).
5. Where a test asserts the sanitized behavior (5xx body carries only generic message), note it as positive evidence.

### Good example
```js
srv.on('error', (err, req) => {
  err.correlationId = req.headers['x-correlation-id'];  // safe augmentation
});  // 5xx bodies stay generic; details are in the logs, keyed by correlation ID
```

### Bad example
```js
srv.on('error', (err, req) => {
  err.message = `${err.message}\n${err.stack}\nquery: ${err.query}`; // internals to client
  err.$sanitize = false;                                              // sanitizer disabled
});
```

### Exception guidance
None for production. Development/test profiles may expose full details freely — the review checks the profile guard, not the intent.

### SAP reference
- https://cap.cloud.sap/docs/node.js/events (5xx generic-message behavior; "should never disclose internal information"; `$sanitize` caution)
- https://cap.cloud.sap/docs/java/event-handlers/indicating-errors (broken error handlers default to generic 500 "not display any error details"; ServiceException messages rendered in OData error responses)

---

## CAP-ERR-003 — Fail loudly — never swallow errors

| Field | Value |
|---|---|
| **Rule ID** | CAP-ERR-003 |
| **Title** | Fail loudly — never swallow errors |
| **Category** | Error handling |
| **Severity** | High |
| **Authority** | SAP-REC (documented "Let It Crash" best practice, strongly worded, on the Node.js page; the same discipline for Java is GEN — no equivalent Java page statement) |
| **Applicability** | All custom code |
| **Runtime** | Both (SAP source is Node.js; Java applicability is general practice, noted per detection) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ERR-001, CAP-INT-007 (remote-failure outcomes must be *defined*, which may include fallbacks — that is decided design, not swallowing), CAP-MT-003 (the multitenant escalation SAP itself names) |
| **Last verified** | 2026-08-12 |

### Rule statement
Errors that code cannot meaningfully handle MUST propagate: SAP's documented practice — "Fail loudly: Do not hide errors and silently continue. Make sure that unexpected errors are correctly logged." and "Do not catch errors you can't handle." When re-throwing, the original error is augmented, not replaced ("Don't Hide Origins of Errors"). Catch blocks that log-and-continue, return default values on unexpected failures, or discard the original exception are prohibited. SAP's own escalation rationale applies: after an unexpected error "you can never be 100% certain that any shared resource wasn't affected… especially in multitenant apps that bear the risk of information disclosure."

### Rationale
A swallowed unexpected error converts a crash (visible, restartable, platform-healed) into undefined state (invisible, compounding): half-applied changes proceed as if successful, and in multitenant apps SAP explicitly names information-disclosure risk. Empty catch blocks are where production integrity quietly dies. **High:** the documented failure mode is continued operation on possibly corrupted shared state — an integrity class, kept below Critical because manifestation requires an actual error plus subsequent damage.

### Implementation guidance
- Catch only where you have a *defined* recovery (CAP-INT-007's designed fallbacks qualify — they're decided outcomes, not suppression).
- Re-throw with context: attach information to the original error (`e.message += …; throw e`), never `throw new Error('failed')` discarding the cause.
- Let the framework's error path do its job — CAP logs, sanitizes (CAP-ERR-002), and rolls back (Java: documented).

### Evidence expected in code
No empty/log-only catch blocks on unexpected-error paths; augmented re-throws preserving originals; defined fallbacks only where designed (comment/design reference).

### Detection guidance
1. Search for empty catch blocks (`catch {}`, `catch (e) {}`) and log-only catches that continue normal flow after unexpected errors → FAIL per site with file:line.
2. Search for default-value returns on generic failures (`catch { return []; }`, `catch { return null; }`) masking outages → FAIL (cross-report CAP-INT-007 where remote calls are involved: distinguish *designed* fallback — referenced config/comment — from silent default).
3. Search re-throw sites discarding originals (`throw new Error('…')` inside catch without cause/augmentation; Java `throw new RuntimeException("failed")` without the cause parameter) → FAIL.
4. Java: same patterns as general practice — report findings with GEN authority note.
5. Verify unexpected-error logging exists on the paths that do catch (framework default suffices when errors propagate).

### Good example
```js
try { await postToLedger(entry); }
catch (e) {
  e.message = `posting ledger entry ${entry.ID}: ${e.message}`;  // augment…
  throw e;                                                        // …and propagate
}
```

### Bad example
```js
try { await postToLedger(entry); }
catch (e) { console.log('ledger issue', e.code); }   // swallowed: request
await markOrderPosted(order);                         // continues, marks as posted —
                                                      // books and ledger now disagree
```

### Exception guidance
Defined-recovery catches (retry per CAP-INT-007, designed fallbacks, cleanup-then-rethrow) are the legitimate pattern — recognizable by producing a *decided* outcome and preserving the original error. Best-effort auxiliary actions (e.g., optional notification) may catch-and-log when the comment says so.

### SAP reference
- https://cap.cloud.sap/docs/node.js/best-practices ("Let It Crash"; "Fail loudly"; "Do not catch errors you can't handle"; multitenant disclosure rationale; "Don't Hide Origins of Errors")

---

## CAP-ERR-004 — Keep Node.js custom error handlers synchronous

| Field | Value |
|---|---|
| **Rule ID** | CAP-ERR-004 |
| **Title** | Keep Node.js custom error handlers synchronous |
| **Category** | Error handling |
| **Severity** | Medium |
| **Authority** | SAP-REQ ("They are expected to be a sync function, that is, not `async`, not returning Promises") |
| **Applicability** | Node.js projects registering `srv.on('error')` handlers; NOT APPLICABLE otherwise |
| **Runtime** | Node.js |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ERR-002 (what such handlers may change), CAP-TXN-006 (general async discipline — this is its deliberate exception: no async at all here) |
| **Last verified** | 2026-08-12 |

### Rule statement
Custom `srv.on('error')` handlers MUST be synchronous functions — not `async`, not returning Promises (documented constraint) — and MUST restrict themselves to augmenting/modifying the outgoing error (their documented purpose). No I/O, no database access, no emits inside error handlers.

### Rationale
The error hook runs at response-rendering time where the framework does not await — an async handler's work races the response: modifications apply never or nondeterministically, and started-but-unawaited operations violate the transaction discipline (CAP-TXN-006) at the worst possible moment. **Medium:** misuse yields nondeterministic error output and stray async work; scoped to projects using the hook.

### Evidence expected in code
`srv.on('error', (err, req) => …)` as plain synchronous functions performing only in-place augmentation.

### Detection guidance
1. Search for `srv.on('error'`/`this.on('error'` registrations.
2. Handler declared `async` or returning a Promise → FAIL with file:line.
3. Handler body performing I/O (queries, emits, HTTP, fs) → FAIL (side effects belong in normal handlers/logging).
4. Handler mutating beyond the error object (writing state) → FAIL.
5. NOT APPLICABLE if no custom error handlers exist.

### Good example
```js
srv.on('error', (err, req) => {          // sync, augmentation only
  if (err.code === 'SQLITE_CONSTRAINT') err.code = 'UNIQUE_VIOLATION';
});
```

### Bad example
```js
srv.on('error', async (err, req) => {                 // async: races the response
  await INSERT.into(ErrorLog).entries({ msg: err.message }); // I/O in the hook
});
```

### Exception guidance
None — needs that exceed augmentation (error persistence, notifications) belong in normal handler flow, logging (CAP-LOG), or queued processing (CAP-EVT-002).

### SAP reference
- https://cap.cloud.sap/docs/node.js/core-services (`srv.on('error')`: sync-only, augment/modify outgoing errors)

---

## CAP-ERR-005 — Localize error messages via stable codes and targets

| Field | Value |
|---|---|
| **Rule ID** | CAP-ERR-005 |
| **Title** | Localize error messages via stable codes and targets |
| **Category** | Error handling |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | Projects with end-user-facing error messages (UIs, localized APIs); NOT APPLICABLE for purely technical APIs with a documented single-locale decision |
| **Runtime** | Both (Node: `i18n/messages` bundles keyed by `code`, honoring `Accept-Language`; Java: `messages.properties` via Spring resource bundles) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-ERR-001 (codes originate there), CAP-CDS-008 (data localization — separate concern), CAP-TEST-003 (why tests assert codes, not texts); org code-taxonomy is gap G-02 |
| **Last verified** | 2026-08-12 |

### Rule statement
User-facing error messages MUST be localized through the framework mechanism — stable message `code`s looked up in i18n bundles (Node.js: "If `code` is given… it is used to look up a user-readable error `message` from the `i18n/messages` bundles", honoring `Accept-Language`; Java: translated texts in `messages.properties`) — not hardcoded English strings in handlers. Field-level errors SHOULD carry a `target` so UIs (SAP Fiori interprets `target` in OData V4 error messages) attach the message to the right control.

### Rationale
Hardcoded message strings bypass translation entirely (a German user gets English validation errors regardless of `Accept-Language`), scatter copy across handlers, and push tests into asserting texts (CAP-TEST-003's anti-pattern). Codes + bundles centralize copy, enable translation, and keep API contracts stable. **Medium:** i18n/API-quality; nothing functional breaks in the default locale — which is exactly why it ships unnoticed.

### Evidence expected in code
`i18n/messages*.properties` bundles (Node) / `src/main/resources/messages*.properties` (Java) containing the error texts; handlers passing codes (and targets), not literal sentences; translations for the supported locales.

### Detection guidance
1. Determine locale requirements (supported languages from requirements/UI setup); single-locale documented decision → NOT APPLICABLE for the translation element (targets element may still apply).
2. Search handlers for literal user-facing message strings in `req.error/reject`/`ServiceException`/`Messages` calls → each → FAIL (Medium) with the code+bundle replacement.
3. Verify bundles exist and contain the codes used; codes referenced but missing from bundles → FAIL (falls back to raw code).
4. Check field-level validation errors carry `target` (Node third arg/object form; Java `.target(...)`) where a UI consumes them → missing → observation/FAIL for Fiori-consumed services.
5. Spot-check one non-default locale bundle for completeness → gaps → observation.

### Good example
```js
srv.before('CREATE', 'Orders', req => {
  if (req.data.quantity > stock) req.error(400, 'ORDER_EXCEEDS_STOCK', 'quantity');
});
```
```properties
# i18n/messages_de.properties
ORDER_EXCEEDS_STOCK = Die Bestellmenge übersteigt den verfügbaren Bestand
```

### Bad example
```js
req.error(400, `Sorry, we only have ${stock} items left!`);  // untranslatable,
// unstable for tests, interpolated user-adjacent values unvalidated (CAP-ERR-006)
```

### Exception guidance
Internal/technical services with a documented single-locale policy skip translation but SHOULD still use stable codes (they cost nothing and serve tests/monitoring). Log messages are not user-facing — English-only is fine there.

### SAP reference
- https://cap.cloud.sap/docs/node.js/events (code → i18n/messages lookup; `Accept-Language`)
- https://cap.cloud.sap/docs/java/event-handlers/indicating-errors (`messages.properties`; Fiori `target` interpretation)

---

## CAP-ERR-006 — Validate user input embedded in messages

| Field | Value |
|---|---|
| **Rule ID** | CAP-ERR-006 |
| **Title** | Validate user input embedded in messages |
| **Category** | Error handling |
| **Severity** | High |
| **Authority** | SAP-REQ (documented on both runtimes: "Ensure proper validation of the message text if it contains values from user input" / Java adds URLs) |
| **Applicability** | All messages (errors, warnings, info) whose text or parameters include user-provided values |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-016 (log-injection twin), CAP-SEC-012 (input validation generally), CAP-ERR-005 (parameterized codes reduce the exposed surface); downgraded from candidate Critical |
| **Last verified** | 2026-08-12 |

### Rule statement
User-provided values embedded in message texts (or message URLs, per the Java documentation) MUST be validated/sanitized before inclusion — SAP, on both runtimes: "Ensure proper validation of the message text [and URL] if they contain values from user input." Prefer parameterized bundle messages (CAP-ERR-005) where the user value is a typed placeholder; raw interpolation of free-text user input into messages is prohibited.

### Rationale
Error messages travel: into OData responses rendered by UIs, into logs (CAP-SEC-016), into support tickets and notification texts. Unvalidated user input inside them is an injection channel — script/markup toward UI consumers, forged entries toward log processing, malicious URLs toward anyone who clicks. **High (downgraded from candidate Critical):** a real injection channel whose exploitation depends on the consumer's rendering/processing — reflected exposure, not direct data access.

### Evidence expected in code
Messages built from codes + typed parameters; free-text user input either absent from messages or passed through validation/escaping; no user-controlled URLs in messages without allow-listing.

### Detection guidance
1. Search message-construction sites (`req.error/reject/notify/info/warn`, `Messages.*`, `ServiceException(...)`) for interpolation of `req.data.*`/user-derived variables into the text → per site: value is typed/constrained (number, enum, validated per CAP-SEC-012) → PASS; free-text user input raw → FAIL with file:line.
2. Check for user-controlled URLs placed into messages (Java-documented case) without allow-list validation → FAIL.
3. Verify the parameterized-bundle pattern is available and used (codes with placeholders) — its absence pushes developers toward interpolation (observation).
4. Cross-check the same values' log treatment under CAP-SEC-016 (one finding can violate both — report in both, fix once).

### Good example
```js
// typed placeholder via bundle: ORDER_EXCEEDS_STOCK = Only {0} items available
req.error(400, 'ORDER_EXCEEDS_STOCK', [stock], 'quantity');   // stock is a number
```

### Bad example
```js
// free-text user input reflected raw into the error message
req.error(400, `Book "${req.data.title}" could not be ordered`);
// title = '<script>…' now travels into every consumer of the error
```

### Exception guidance
Typed, framework-validated values (numbers, dates, enum members, `@assert.format`-constrained fields) are safe placeholders by construction — that *is* the compliant pattern, not an exception. No exception for raw free-text reflection.

### SAP reference
- https://cap.cloud.sap/docs/node.js/events ("Ensure proper validation of the message text if it contains values from user input")
- https://cap.cloud.sap/docs/java/event-handlers/indicating-errors (same warning incl. URLs)
