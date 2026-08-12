# CAP-LOG — Logging & observability

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-03 (log content/retention policy), G-06 (authorization audit trail), G-37 (alert thresholds/probe intervals).

**Rules:** 5 active (0 Critical, 1 High, 4 Medium). All SAP references verified against official CAP documentation on **2026-08-12**.

Scope boundaries: **sensitive-data hygiene in logs is [CAP-SEC-016](security.md)** (secrets/PII, masking, injection, production levels) — not restated here. **Application logs are not audit logs:** audit logging of personal-data access is [CAP-PRIV-002](data-privacy.md) (`@cap-js/audit-logging` / SAP Audit Log service, see the [source map](../../references/sap-cap-sources.md) §6); CAP documents **no** automatic authorization-decision logging (ORG gap G-06); retention policy is ORG (G-03). Health-check wiring is [CAP-OPS-001](production-readiness.md); monitoring/alerting mandates remain ORG (G-37) — no monitoring SLAs are invented here.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-LOG-001 | Log through the framework's logging facilities with per-component levels | Medium | SAP-REC | Both |
| CAP-LOG-002 | Emit structured JSON logs in production | Medium | SAP-REC | Both |
| CAP-LOG-003 | Preserve correlation-ID propagation | Medium | SAP-REC | Both |
| CAP-LOG-004 | Use SAP-provided OpenTelemetry tooling when adopting telemetry | Medium | SAP-REC | Both |
| CAP-LOG-005 | Expose only the health actuator publicly | High | SAP-REC | Java |

---

## CAP-LOG-001 — Log through the framework's logging facilities with per-component levels

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOG-001 |
| **Title** | Log through the framework's logging facilities with per-component levels |
| **Category** | Logging & observability |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | All application logging |
| **Runtime** | Both (Node.js: `cds.log`; Java: SLF4J via Spring Boot) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-016 (what may be logged; masking; production levels — owned there), CAP-LOG-002; absorbs candidate CAP-LOG #4 (level configuration) |
| **Last verified** | 2026-08-12 |

### Rule statement
Application code MUST log through the framework facilities — Node.js: component loggers via `cds.log('<component>')` (never `console.*` in `srv/**`); Java: SLF4J with **parameterized** messages (SAP: "Prefer passing parameters over concatenating the message" — concatenation builds the string regardless of level). Log levels MUST be configured per component/module (Node: `cds.log.levels` / `DEBUG=<modules>`; Java: `logging.level.*` per logger), not by raising a global level; production log-level hygiene (no global DEBUG, masking intact) is enforced by CAP-SEC-016.

### Rationale
Component loggers are what make selective diagnostics possible ("projects can set different log levels for different components/layers") and what the JSON formatter (CAP-LOG-002), masking, and correlation enrichment hook into — `console.log` output bypasses all of it. Parameterized Java logging avoids paying string-construction costs for suppressed levels. **Medium:** operational quality; the security-relevant aspects are CAP-SEC-016's.

### Evidence expected in code
`cds.log('<component>')` loggers per module (Node); SLF4J loggers with `{}` placeholders (Java); per-component level configuration; no `console.*` in server code.

### Detection guidance
1. Node: search `srv/**` for `console.log|warn|error|info` → each → FAIL with file:line (name the `cds.log` replacement).
2. Node: verify component IDs are meaningful (per module/feature), enabling per-component levels; single anonymous logger everywhere → observation.
3. Java: search log statements for string concatenation with variables (`"…" + value`) → FAIL per site (parameterized form).
4. Check level configuration exists per component where non-default levels are used (`cds.log.levels`, `logging.level.com.acme.*`) rather than a global override → global-only tuning → observation; production DEBUG globals → report under CAP-SEC-016.
5. Report representative sites.

### Good example
```js
const LOG = cds.log('ordering');            // component logger
LOG.info('order created', { id: order.ID });
```
```java
logger.info("Consolidating order {}", order.getId());   // parameterized
```

### Bad example
```js
console.log('DEBUG order = ' + JSON.stringify(order));  // bypasses formatter,
                                                          // masking, levels
```

### Exception guidance
CLI scripts and build tooling (outside the served application) may use `console`. Temporary diagnostic output must not survive into commits (review catches it here).

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-log (component loggers; per-module levels; `DEBUG` env)
- https://cap.cloud.sap/docs/java/operating-applications/observability (SLF4J; prefer parameters over concatenation; per-logger levels)

---

## CAP-LOG-002 — Emit structured JSON logs in production

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOG-002 |
| **Title** | Emit structured JSON logs in production |
| **Category** | Logging & observability |
| **Severity** | Medium |
| **Authority** | SAP-REC (Node.js: documented **default** behavior to preserve — "The JSON log formatter is the default formatter in production"; Java: documented setup via `cf-java-logging-support` with Spring profiles) |
| **Applicability** | Production configurations |
| **Runtime** | Both — asymmetric: Node.js gets JSON by default (since `@sap/cds` 7.5, no extra config); Java requires adding `cf-java-logging-support` with a cloud Spring profile |
| **CAP version** | ⏱ Node default since `@sap/cds` 7.5 (still stated on current docs) |
| **Status** | Active |
| **Related rules** | CAP-LOG-001, CAP-LOG-003 (JSON formatter carries `correlation_id`), CAP-SEC-016 |
| **Last verified** | 2026-08-12 |

### Rule statement
Production logs MUST be structured (JSON) so the platform log services can parse, index, and correlate them: Node.js projects MUST NOT disable the production-default JSON formatter; CAP Java projects MUST set up JSON logging via `cf-java-logging-support` with profile-specific logger configuration (SAP's documented pattern: `<springProfile name="cloud">` vs local) — plain-text production logs are non-compliant on either runtime.

### Rationale
The platform stack (SAP Cloud Logging / Application Logging) operates on structured fields — correlation IDs, categories, custom fields, tenant metadata ride the JSON formatter. Plain-text production logs reduce incident diagnostics to grep-and-hope: no correlation filtering, no cross-instance tracing. The Node side is compliance-by-default (the rule guards against disabling it); the Java side is an active setup step routinely forgotten. **Medium:** diagnostics capability, discovered missing exactly during incidents.

### Evidence expected in code
Node: no formatter override to plain text in production profiles. Java: `cf-java-logging-support-logback` (or equivalent documented library) in `pom.xml` plus a `logback-spring.xml` with cloud/local profile split.

### Detection guidance
1. Node: search config for log-formatter overrides (`cds.log.format`, custom formatters) active in production → plain-text override → FAIL.
2. Java: check `pom.xml` for the cf-java-logging-support dependency and `src/main/resources/logback-spring.xml` for the `springProfile` split → both absent in a production-targeted project → FAIL.
3. Verify the deployment's Spring profile activates the JSON configuration in cloud deployments (mta.yaml/chart env: `SPRING_PROFILES_ACTIVE=cloud`) → mismatch → FAIL.
4. If deployed-log samples are available, one structured log line is positive evidence; otherwise static config suffices.
5. NOT APPLICABLE only for projects without a production target.

### Good example
```xml
<!-- logback-spring.xml — SAP's documented profile split -->
<springProfile name="cloud">
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="com.sap.hcp.cf.logback.encoder.JsonEncoder"/>
  </appender>
</springProfile>
<springProfile name="!cloud">
  <!-- human-readable local pattern -->
</springProfile>
```

### Bad example
```text
Java production app with default Logback pattern layout — multi-line
plain text in the log service; correlation filtering and tenant
attribution impossible during the first real incident.
```

### Exception guidance
None for production platform deployments. Purely local/edge deployments with their own log pipeline may document an alternative structured format (the *structured* requirement stands; JSON via the platform libraries is the default path).

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-log ("The JSON log formatter is the default formatter in production"; since `@sap/cds` 7.5)
- https://cap.cloud.sap/docs/java/operating-applications/observability (`cf-java-logging-support` with Spring-profile-specific configuration)

---

## CAP-LOG-003 — Preserve correlation-ID propagation

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOG-003 |
| **Title** | Preserve correlation-ID propagation |
| **Category** | Logging & observability |
| **Severity** | Medium |
| **Authority** | SAP-REC (the propagation itself is documented automatic behavior on both runtimes; the rule forbids breaking it) |
| **Applicability** | All projects; particularly custom middleware/filters and outbound calls |
| **Runtime** | Both (Node: header array → `cds.context.id`, JSON formatter maps to `correlation_id`; Java: "handled out of the box", `X-CorrelationID` accepted/forwarded, thread propagation via Request Contexts, propagation to remote services via Cloud SDK) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-LOG-002 (the formatter carries the ID), CAP-INT-002 (framework consumption keeps propagation), CAP-MT-006 (context in async work), CAP-ERR-002 (correlation ID as the safe link between generic client errors and detailed logs) |
| **Last verified** | 2026-08-12 |

### Rule statement
The framework's automatic correlation handling MUST be preserved end to end: inbound IDs accepted (Node checks the documented header set and ensures `cds.context.id`; Java accepts/forwards `X-CorrelationID` out of the box), included in every log line (via CAP-LOG-002's formatter), and propagated on outbound calls (automatic for framework-consumed remote services — one more reason for CAP-INT-002). Custom code MUST NOT strip, overwrite, or fail to forward correlation headers — in custom middleware, hand-rolled HTTP clients, or thread hand-offs outside the framework's context facilities.

### Rationale
The correlation ID is the thread that stitches one user action across instances, services, and log streams — and the safe reference that lets a sanitized client error (CAP-ERR-002) be matched to its detailed server log. The framework does everything automatically; only custom code breaks it, and every documented breakage pattern in this rule corresponds to a bypass of another rule's mechanism (hand-rolled clients: CAP-INT-002; raw threads: CAP-MT-006). **Medium:** diagnostics capability.

### Evidence expected in code
No custom code generating fresh correlation IDs mid-request or dropping headers on outbound calls; custom middleware passing headers through; async work via framework context facilities.

### Detection guidance
1. Search custom middleware/filters (`server.js` Express middleware, Java servlet filters) for header manipulation dropping/replacing correlation headers (`x-correlation-id`, `X-CorrelationID`, `traceparent`) → FAIL per site.
2. Inspect hand-rolled outbound HTTP clients (flagged by CAP-INT-002): correlation header forwarded? → not forwarded → FAIL here (in addition to the CAP-INT-002 finding).
3. Search for fresh ID generation used in logs instead of `cds.context.id`/the framework's ID → FAIL (splits the trace).
4. Java: raw `Thread`/executor hand-offs outside Request Context facilities lose the ID → report under CAP-MT-006 with a correlation note here.
5. Positive evidence: a test or logged sample showing the same correlation ID across request → handler log → outbound call.

### Good example
```js
const LOG = cds.log('ordering');
LOG.info('payment initiated', { order: order.ID });
// correlation_id lands in the JSON line automatically — nothing to do
```

### Bad example
```js
// custom client drops the header; logs use a private ID —
// the trace dies at the service boundary
const res = await fetch(url, { headers: { 'x-request-id': crypto.randomUUID() } });
LOG.info('called billing', { myTraceId: privateId });
```

### Exception guidance
Systems that cannot accept correlation headers (fixed third-party APIs) break propagation externally — log the outbound correlation ID *with* the call so the trace survives internally. No exception for dropping IDs inside the application.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-log (header acceptance → `cds.context.id`; formatter mapping to `correlation_id`/`w3c_traceparent`)
- https://cap.cloud.sap/docs/java/operating-applications/observability ("handled out of the box"; `X-CorrelationID` accepted and forwarded; thread + remote propagation)

---

## CAP-LOG-004 — Use SAP-provided OpenTelemetry tooling when adopting telemetry

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOG-004 |
| **Title** | Use SAP-provided OpenTelemetry tooling when adopting telemetry |
| **Category** | Logging & observability |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented tooling; adoption itself is not mandated by SAP — whether/what to monitor is ORG territory, G-37) |
| **Applicability** | Projects adopting tracing/metrics instrumentation; NOT APPLICABLE where none is adopted (the *decision* to adopt is M9-gate/ORG territory — CAP-OPS-003/G-37 — not this rule) |
| **Runtime** | Both (Node.js: `@cap-js/telemetry` plugin — automatic OTel instrumentation, exporters incl. SAP Cloud Logging, Dynatrace, Jaeger, OTLP; Java: `cds add cloud-logging --with-telemetry`, assumes the SAP Java Buildpack) |
| **CAP version** | Current plugin/tooling state per docs — no version-status claims beyond that (re-verify at adoption time) |
| **Status** | Active |
| **Related rules** | CAP-LOG-002/-003 (logs/correlation), CAP-SRV-002/CAP-ARCH-002 (no hand-rolled cross-cutting instrumentation layers); alert thresholds/monitoring SLAs are ORG gap G-37 |
| **Last verified** | 2026-08-12 |

### Rule statement
Where a project adopts tracing/metrics, it MUST use the SAP-provided OpenTelemetry integrations — Node.js: the `@cap-js/telemetry` plugin ("automatic OpenTelemetry instrumentation") with its documented exporters; Java: the documented setup via `cds add cloud-logging --with-telemetry` on the SAP Java Buildpack — rather than hand-wiring OTel SDKs through custom interceptor layers around CAP internals. What to monitor and which thresholds to alert on remain ORG decisions (G-37) — this rule governs only the *mechanism* once telemetry is adopted.

### Rationale
The SAP integrations instrument CAP's own layers (request handling, service calls, database interaction) with correct context — a hand-wired OTel setup wrapping CAP services re-creates exactly the wrapper-layer problem CAP-ARCH-002 prohibits, and typically misses tenant/correlation coupling. CAP-native-first, applied to observability. **Medium:** mechanism quality for an optional capability.

### Evidence expected in code
If telemetry exists: `@cap-js/telemetry` in dependencies with exporter config (Node) or the cloud-logging/telemetry facet artifacts (Java); no bespoke OTel wiring wrapping CAP services.

### Detection guidance
1. Determine whether telemetry is adopted (dependencies: `@cap-js/telemetry`, `@opentelemetry/*`; Java OTel agent config) → none → NOT APPLICABLE (note the adoption question for the CAP-OPS gate).
2. Adopted via the SAP tooling → PASS; verify exporter config targets a supported backend (Cloud Logging/Dynatrace/Jaeger/OTLP).
3. Raw `@opentelemetry/*` SDK wiring instrumenting CAP services through custom wrappers → FAIL (name the plugin path; custom *additional* spans inside handlers are fine).
4. Java: verify the buildpack assumption holds for agent-based setup (deployment uses SAP Java Buildpack) → mismatch → observation with the documented constraint.
5. Report configuration locations.

### Good example
```jsonc
// package.json — SAP plugin, documented exporter
{ "dependencies": { "@cap-js/telemetry": "^1" },
  "cds": { "requires": { "telemetry": { "kind": "telemetry-to-cloud-logging" } } } }
```

### Bad example
```js
// bespoke OTel wiring wrapping every CAP service — a custom
// instrumentation framework around the framework (CAP-ARCH-002)
const services = ['CatalogService','AdminService'];
for (const s of services) wrapWithSpans(await cds.connect.to(s), tracer);
```

### Exception guidance
Custom spans/metrics *inside* handlers (business KPIs) via the standard OTel API are complementary and fine. Platforms with mandated third-party agents (org APM standard) may run those agents alongside — documented per CAP-ARCH-007.

### SAP reference
- https://cap.cloud.sap/docs/plugins/ (Telemetry plugin: automatic OpenTelemetry instrumentation)
- https://cap.cloud.sap/docs/java/operating-applications/observability (`cds add cloud-logging --with-telemetry`; SAP Java Buildpack assumption)

---

## CAP-LOG-005 — Expose only the health actuator publicly

| Field | Value |
|---|---|
| **Rule ID** | CAP-LOG-005 |
| **Title** | Expose only the health actuator publicly |
| **Category** | Logging & observability |
| **Severity** | High |
| **Authority** | SAP-REC, security-motivated (verified wording: "For security reasons, it's recommended to expose only the `health` actuator as web endpoint. All other actuators are recommended for local JMX-based access.") |
| **Applicability** | CAP Java projects using Spring Boot actuators; NOT APPLICABLE for Node.js (no actuator mechanism — the built-in `/health` endpoint is the only such surface) |
| **Runtime** | Java |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-015 (backend exposure verification), CAP-SEC-017 (what env-style actuators would leak); health-check *wiring* is [CAP-OPS-001](production-readiness.md) |
| **Last verified** | 2026-08-12 |

### Rule statement
CAP Java deployments MUST expose only the `health` actuator as a web endpoint — all other Spring Boot actuators (env, beans, heapdump, loggers, mappings, …) stay on local/JMX access per SAP's security-motivated recommendation. Where detailed health output or further actuators are genuinely needed over HTTP, they MUST be protected by authentication/authorization (and that protection verified per CAP-SEC-015's unauthenticated-rejection testing).

### Rationale
Non-health actuators are an operational backdoor: `/actuator/env` exposes configuration (potentially credentials), `/actuator/heapdump` exports process memory, `/actuator/loggers` allows runtime reconfiguration. SAP words the restriction explicitly "for security reasons". **High justification (above the REC wording):** a publicly exposed env/heapdump actuator is directly exploitable information disclosure up to credential capture — the severity reflects violation impact, not SAP's phrasing; kept below Critical because exploitation requires the actuator to actually carry sensitive material and the endpoint to be reachable (both common, neither universal).

### Evidence expected in code
`management.endpoints.web.exposure.include: health` (or equivalent restrictive config) in `application*.yaml`; no broad exposure (`include: "*"`); any additional web-exposed actuator wrapped in security configuration.

### Detection guidance
1. Inspect `application*.yaml`/properties for `management.endpoints.web.exposure` → `include: "*"` or listing non-health actuators without security config → FAIL with file:line.
2. No explicit config → determine the effective Spring Boot default for the version in use and whether the deployment route exposes `/actuator/**` → report accordingly (default-only exposure of health → PASS).
3. Where additional actuators are web-exposed: verify Spring Security protects them (config + a denial test per CAP-TEST-007/CAP-SEC-015) → unprotected → FAIL.
4. Check the deployed route/gateway doesn't separately expose management ports publicly (mta.yaml/chart) → NOT ASSESSABLE if platform config is outside the repo — name the needed evidence.
5. NOT APPLICABLE for Node.js projects.

### Good example
```yaml
# application.yaml — SAP's documented posture
management:
  endpoints:
    web:
      exposure:
        include: health
```

### Bad example
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"    # env, heapdump, loggers… publicly reachable
```

### Exception guidance
Platform-internal monitoring needing more actuators uses network-internal routes or authenticated access — documented, with the protection test as evidence. No exception for unauthenticated public exposure of non-health actuators.

### SAP reference
- https://cap.cloud.sap/docs/java/operating-applications/observability ("For security reasons, it's recommended to expose only the `health` actuator as web endpoint…")
