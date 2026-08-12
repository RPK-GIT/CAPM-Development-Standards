# CAP-OPS — Production readiness & operations

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gaps: G-36 (made concrete by CAP-OPS-003), G-37 (alert thresholds/probe intervals — still open).

**Rules:** 3 active (0 Critical, 1 High, 2 Medium). All SAP references verified against official CAP documentation on **2026-08-12**.

Scope boundaries: observability *mechanics* are [CAP-LOG](logging-observability.md) (structured logs, correlation, telemetry, actuator exposure); dead-letter operations are [CAP-EVT-005](events-messaging.md); tenant-upgrade operations are [CAP-MT-005](multitenancy.md); version currency is [CAP-VER-002](versions-dependencies.md). No monitoring SLAs, alert thresholds, or scaling values are invented (G-37/G-04). Candidate CAP-OPS #3 (MCP governance) was merged into [CAP-SEC-018](security.md) in Batch 1 and is not recreated; candidate #4 (topology-by-configuration) merged into [CAP-ARCH-006](architecture.md).

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-OPS-001 | Wire health probes into the deployment descriptors | Medium | SAP-REC | Both |
| CAP-OPS-002 | Ship a real production UI entry point | Medium | SAP-REC | Both |
| CAP-OPS-003 | Pass the M9 production-readiness assessment before go-live | High | ORG | Both |

---

## CAP-OPS-001 — Wire health probes into the deployment descriptors

| Field | Value |
|---|---|
| **Rule ID** | CAP-OPS-001 |
| **Title** | Wire health probes into the deployment descriptors |
| **Category** | Production readiness & operations |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented endpoints and facet-generated wiring; deviations "adjust the values created by `cds add`" — documented duty) |
| **Applicability** | All deployed applications |
| **Runtime** | Both — Node.js: built-in `/health` ("an out-of-the-box endpoint for liveness and readiness checks", answering 200 `{ status: 'UP' }`); Java: Spring Boot actuator probes `/actuator/health/liveness` and `/actuator/health/readiness`, configured by the `mta` and `kyma` facets |
| **CAP version** | ⏱ Current facets wire the probes; documented CF tooling caveat: readiness checks "not yet supported by the Cloud Foundry CLI as well as the Cloud MTA Build Tool" — re-verify at releases |
| **Status** | Active |
| **Related rules** | CAP-DEP-001/-003 (the descriptors), CAP-LOG-005 (only health exposed publicly — Java), CAP-SEC-015 (health is the deliberate public exception) |
| **Last verified** | 2026-08-12 |

### Rule statement
Deployed applications MUST have health probes wired in their deployment descriptors so the platform can manage their lifecycle: Node.js apps use the built-in `/health` endpoint; Java apps the actuator liveness/readiness probes — both wired automatically by the `mta`/`kyma` facets (CAP-DEP-001/-003). Projects using a custom `server.js` or custom health endpoints MUST adjust the generated descriptor values accordingly (documented duty). The documented CF tooling limitation (readiness checks not yet supported by CF CLI/MBT) is acknowledged — on CF, the achievable configuration is applied and the gap noted in the runbook rather than papered over.

### Rationale
Health probes are how the platform distinguishes "starting", "alive", and "ready for traffic": without them, Kubernetes routes requests to pods that aren't ready and keeps hung instances in rotation; CF falls back to coarse port checks. The facets generate correct wiring — the failure mode is custom servers/endpoints silently orphaning the generated configuration. **Medium:** degraded lifecycle management and restart behavior; loud only under failure conditions (which is when it matters). Exposure hygiene of the endpoints themselves is CAP-LOG-005/CAP-SEC-015.

### Evidence expected in code
Probe configuration in `mta.yaml`/Helm values matching the actual endpoints; for custom servers: adjusted descriptor values; runbook note for the CF readiness-tooling gap where applicable.

### Detection guidance
1. Inspect deployment descriptors: health-check configuration present (facet-generated readiness/liveness in the chart; health-check settings in `mta.yaml`) → absent for a deployed app → FAIL.
2. Verify endpoint agreement: configured paths match reality — Node `/health` (or the custom server's actual endpoint), Java actuator paths enabled (`management.endpoint.health.probes` / facet defaults) → mismatch (custom `server.js` with stale generated values) → FAIL (documented adjust-duty).
3. Kyma: liveness *and* readiness in the chart values → missing → FAIL.
4. CF: apply what the tooling supports; verify the runbook acknowledges the documented readiness gap → silently assumed readiness support → observation.
5. Cross-check public exposure under CAP-LOG-005 (Java: only health public).

### Good example
```yaml
# chart/values.yaml (kyma facet) — probes match the served endpoints
srv:
  health:
    liveness:  { path: /actuator/health/liveness }
    readiness: { path: /actuator/health/readiness }
```

### Bad example
```text
Custom server.js moved everything under /api and disabled the default
middlewares — the mta.yaml still points health checks at /health, which
now 404s; the platform restarts healthy instances in a loop.
```

### Exception guidance
Batch-style/worker modules without HTTP interfaces use the platform's process-level checks — note it in the descriptor. No exception for request-serving modules.

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/health-checks (Node `/health` liveness+readiness; Java actuator probes wired by `mta`/`kyma` facets; CF CLI/MBT readiness caveat; adjust values for custom servers/endpoints)

---

## CAP-OPS-002 — Ship a real production UI entry point

| Field | Value |
|---|---|
| **Rule ID** | CAP-OPS-002 |
| **Title** | Ship a real production UI entry point |
| **Category** | Production readiness & operations |
| **Severity** | Medium |
| **Authority** | SAP-REC (documented: the dev index page and Fiori preview "aren't available in the cloud"; "you should add a proper SAP Fiori elements application" via the documented UI options) |
| **Applicability** | Applications with end-user UIs; NOT APPLICABLE for headless/API-only services (state the consumer) |
| **Runtime** | Both |
| **CAP version** | Current UI facets: `approuter` (custom), `workzone` (managed approuter/launchpad), `portal` (multitenant), `app-frontend` |
| **Status** | Active |
| **Related rules** | CAP-DEP-001 (the UI module in the MTA), CAP-SEC-015 (the approuter is *not* the security boundary — backends still authenticate) |
| **Last verified** | 2026-08-12 |

### Rule statement
Applications with end users MUST ship a real UI entry point in production — via one of the documented options (`cds add approuter`, `workzone`, `portal` for multitenant scenarios, or `app-frontend`) — because the development index page and Fiori preview "are meant only for the development profile and aren't available in the cloud." Relying on dev-time pages as the de-facto UI is a go-live blocker discovered at go-live.

### Rationale
Teams develop against the generated index page and Fiori preview for months; in the cloud those pages simply don't exist, and "how do users reach the app?" becomes an unplanned workstream during the production push. The documented facets wire the entry point (approuter routing, launchpad integration) into the deployment structure ahead of time. **Medium:** a readiness gap, loud at exactly the wrong moment; no security or integrity impact by itself (the approuter's non-role as a security boundary is CAP-SEC-015).

### Evidence expected in code
A UI module/facet in the deployment (approuter module, workzone/portal artifacts, or app-frontend setup) routing to the application's UIs; go-live docs naming the entry URL.

### Detection guidance
1. Determine whether end users exist (UI apps in `app/`, requirements) → headless/API-only → NOT APPLICABLE (name the consuming systems).
2. Check the deployment for a UI entry option: approuter module in `mta.yaml`/chart, workzone/portal configuration, or app-frontend artifacts → none while `app/` UIs exist → FAIL.
3. Verify the entry point actually routes to the deployed UIs (approuter `xs-app.json` routes / launchpad config referencing the apps) → dangling config → FAIL.
4. Cross-check backend authentication independence under CAP-SEC-015 (not here).

### Good example
```text
mta.yaml: bookshop-approuter module (cds add approuter) routing to
app/browse and app/admin; go-live runbook names the entry URL and
role collections per UI.
```

### Bad example
```text
Demo always ran on the cds watch index page. Production go-live day:
the URL serves 404 — there is no index page in the cloud, and no
approuter/launchpad was ever set up.
```

### Exception guidance
API-only products (consumed by other systems or a customer-owned frontend) are the NOT APPLICABLE path — the review names the consumer instead. Embedded UIs served by a customer's own gateway are a documented deviation (ADR).

### SAP reference
- https://cap.cloud.sap/docs/guides/deploy/to-cf (dev index/Fiori preview unavailable in the cloud; "should add a proper SAP Fiori elements application"; UI facet options incl. `app-frontend`)

---

## CAP-OPS-003 — Pass the M9 production-readiness assessment before go-live

| Field | Value |
|---|---|
| **Rule ID** | CAP-OPS-003 |
| **Title** | Pass the M9 production-readiness assessment before go-live |
| **Category** | Production readiness & operations |
| **Severity** | High |
| **Authority** | ORG (our go-live policy — SAP publishes no consolidated production-readiness checklist; this rule closes gap G-36 by binding the [M9 lifecycle gate](../../development/lifecycle.md)) |
| **Applicability** | Every production go-live (initial, and major-change re-assessments per the lifecycle) |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | This rule *aggregates* — its elements are owned by: CAP-SEC (full Critical/High re-check), CAP-MT-005, CAP-EVT-005 (dead-letter runbook), CAP-LOG-001…005, CAP-OPS-001/-002, CAP-VER-001/-002/-003, CAP-CICD-001/-002/-003, CAP-PRIV-002/-004, CAP-DB-001/-007; process per [review-model](../../reviews/review-model.md) |
| **Last verified** | 2026-08-12 (ORG policy — no SAP claim to verify) |

### Rule statement
No CAP application goes live without a completed, documented **M9 production-readiness assessment**: a full standard review ([review model](../../reviews/review-model.md)) across all applicable categories with zero unresolved Critical findings and all High findings resolved or formally excepted, **plus** the operational readiness elements verified: observability wired and demonstrated (a request traced end-to-end through structured logs by correlation ID — CAP-LOG-002/-003), health probes live (CAP-OPS-001), UI entry point serving (CAP-OPS-002), the operations runbook present (incl. dead-letter procedure CAP-EVT-005, tenant-upgrade sequence CAP-MT-005 for MT apps, rollback approach CAP-CICD-002, upgrade posture CAP-VER-002), audit logging verified in production configuration where personal data is processed (CAP-PRIV-002), and the pipeline gates active (CAP-CICD-003). The assessment record (review report + [milestone checklist](../../templates/milestone-checklist-template.md)) is the go-live evidence; its absence blocks go-live regardless of the application's actual quality.

### Rationale
SAP documents production preparation piecemeal (facets, health checks, lockfiles) but publishes no consolidated go-live checklist — gap G-36. This standard's M9 gate is our answer: it converts "we think it's ready" into a reviewable artifact by aggregating the catalog's production-relevant rules into one assessment with defined pass criteria. The rule deliberately owns *aggregation and evidence*, not new requirements — every element cites its owning rule, so nothing is double-normed. **High justification:** go-live without the assessment means production entry with unverified security posture, unverifiable operations, and no evidence trail — the gate exists precisely because each individual gap hides until it costs; not Critical because the rule enforces *verification*, and the catastrophic conditions it screens for carry their own Critical ratings where they occur.

### Implementation guidance
- Run the M9 review as scoped in the [lifecycle](../../development/lifecycle.md): full-catalog review + operational drill (trace a request; probe the health endpoints; fire a test alert if alerting exists per ORG policy G-37).
- Use the milestone checklist template as the assessment record; attach the review report and the runbook.
- Re-assess on major changes (new tenancy model, new deployment target, major CAP upgrade — per lifecycle re-gating).

### Evidence expected in code
A completed M9 milestone checklist + standard review report for the go-live (in-repo or linked from it); the runbook; the operational-drill evidence (trace excerpt, probe outputs).

### Detection guidance
1. For a production-deployed (or go-live-pending) application: locate the M9 assessment record (checklist + review report) → absent for a live system → FAIL (High).
2. Verify the review's scope and verdicts: all applicable categories assessed; zero open Critical; High findings resolved or carrying approved exceptions (AI-DOC-002 records) → open Critical/unexcepted High at go-live → FAIL.
3. Verify the operational elements' evidence: runbook exists and covers dead-letter/rollback/(tenant-upgrade); trace-through evidence present; probe/UI checks recorded → missing element → FAIL per element (cite the owning rule).
4. For long-live systems: check re-assessment after major changes per the lifecycle → skipped re-gating → FAIL.
5. NOT ASSESSABLE only where the assessment is claimed to exist outside the repository — name the record needed.

### Good example
```text
docs/milestones/M9-2026-08-checklist.md — completed, signed off;
attached: review report (0 Critical, 2 High excepted per ADR-0033),
runbook v1.3 (dead-letter §4, rollback §6, tenant upgrade §7),
trace evidence: correlation e4f1… followed through 3 services.
```

### Bad example
```text
Go-live happened "because the sprint ended". No review report, no
runbook, no trace evidence; the first dead-letter incident is being
debugged in production by whoever wrote the emitter.
```

### Exception guidance
Pilot deployments explicitly marked non-production (no real users/data) may defer M9 — with the marking verifiable and a date attached. Organizations with an equivalent release-governance process may substitute it if it demonstrably covers the same elements (documented mapping).

### SAP reference
None normative (authority: ORG — closes gap G-36). The aggregated elements cite their SAP sources in their owning rules; related SAP reading: https://cap.cloud.sap/docs/guides/deploy/to-cf (SAP's piecemeal production-preparation steps).
