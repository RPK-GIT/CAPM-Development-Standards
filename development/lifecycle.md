# Milestone-Based Development Lifecycle

Every CAP application governed by this standard progresses through the milestones below. Each milestone ends with a **quality gate** following the same pattern:

```
DEVELOP → SELF-VALIDATE → TEST → CAPM STANDARD REVIEW → REMEDIATE → PASS MILESTONE
```

- **DEVELOP** — per [development-model.md](development-model.md) and the `AI-DEV` rules (when Claude Code implements).
- **SELF-VALIDATE** — the implementer checks the increment against the milestone's applicable rule categories (AI-DEV-008).
- **TEST** — the relevant automated tests exist and pass (`CAP-TEST` rules, `AI-TEST` rules).
- **CAPM STANDARD REVIEW** — an independent review per [reviews/review-model.md](../reviews/review-model.md), scoped to the milestone's categories.
- **REMEDIATE** — FAIL findings are fixed (or exceptions approved and documented per AI-DOC-002), then failed rules are re-reviewed.
- **PASS MILESTONE** — exit criteria met; result recorded.

Milestones may overlap in iterative delivery (e.g., per feature), but **no milestone gate may be skipped** on the path to production. Severity gating: Critical findings always block; High findings block unless a documented exception is approved; Medium must be resolved or accepted by M9; Low is recorded (see [severity scale](../docs/standard-architecture.md#severity-scale)).

> **Phase note:** rule categories are named below; the concrete per-milestone rule checklists are generated in Phase 3, once the Phase 2 rule catalog is authored.

---

## M0 — Requirements

| Aspect | Content |
|---|---|
| **Deliverables** | Requirements document: business capabilities, domain glossary, actors/roles, external systems, NFRs (performance, availability, compliance/privacy, tenancy), explicit out-of-scope list |
| **Applicable standards** | `AI-DEV-003` (restate & surface ambiguity); authority-labelling per [authority levels](../docs/authority-levels.md) |
| **Development activities** | Elicit and document requirements; identify personal data (privacy scope); identify tenancy model; identify integration partners |
| **Validation activities** | Requirements walkthrough; every requirement testable and uniquely identifiable; assumptions and open questions listed |
| **Tests** | None (acceptance criteria drafted per requirement) |
| **Review criteria** | Completeness of NFRs; privacy/tenancy/integration explicitly decided or explicitly deferred |
| **Exit criteria** | Requirements approved; acceptance criteria exist; open questions have owners |

## M1 — Architecture

| Aspect | Content |
|---|---|
| **Deliverables** | Architecture description: runtime choice (Node.js/Java) with rationale, service cut, persistence targets, protocols, integration approach, eventing approach, tenancy design, deployment target (CF/Kyma), security architecture (IdP, auth model); ADRs for each material decision |
| **Applicable standards** | `CAP-ARCH`, `CAP-VER` (version baseline), `CAP-MT` (if multitenant), `CAP-DEP` (target choice); `AI-DEV-002/-010` |
| **Development activities** | Choose runtime, versions, project layout (`cds init` conventions); map requirements to CAP capabilities (CAP-native-first); define module/service boundaries |
| **Validation activities** | Check each requirement has a CAP-capability mapping or a justified custom approach; version baseline against [version-management.md](../docs/version-management.md) |
| **Tests** | None (walking-skeleton build may be used to de-risk) |
| **Review criteria** | Architecture uses CAP-native capabilities; no unjustified custom frameworks; versions current and supported |
| **Exit criteria** | Architecture and ADRs approved; project skeleton builds |

## M2 — Domain / CDS Model

| Aspect | Content |
|---|---|
| **Deliverables** | `db/` CDS domain model: entities, associations/compositions, common aspects (`cuid`, `managed`, …), localized/temporal data where needed, `@PersonalData` annotations for personal data; initial test data |
| **Applicable standards** | `CAP-CDS`, `CAP-PRIV` (model-level), `CAP-PERF` (model-level, e.g. key design, calculated-field placement) |
| **Development activities** | Model the domain (domain-first, normalized, per SAP domain-modeling guide); reuse `@sap/cds/common`; annotate personal data |
| **Validation activities** | `cds compile` clean; model review against naming and modeling rules; privacy annotation completeness vs M0 personal-data inventory |
| **Tests** | Deployment smoke test to dev database (`cds deploy`); model-level assertions where applicable |
| **Review criteria** | Compositions vs associations correct; managed aspects used instead of hand-built audit fields; no premature denormalization; keys per modeling guidance |
| **Exit criteria** | Model compiles and deploys; model review PASS; privacy annotations complete |

## M3 — Services / API

| Aspect | Content |
|---|---|
| **Deliverables** | `srv/` service definitions: projections, actions/functions, protocol choices, input-validation annotations (`@assert.*`, `@mandatory`, `@readonly`), draft-enablement decisions, API documentation |
| **Applicable standards** | `CAP-SRV`, `CAP-SEC` (exposure — no unprotected services), `CAP-PERF` (pagination limits) |
| **Development activities** | Define use-case-oriented services as projections; declarative validation before code; decide protocols; restrict exposure to what consumers need |
| **Validation activities** | Service model compiles; every exposed entity/action deliberate; validation annotations in place |
| **Tests** | API tests for generic CRUD exposure and validation behavior (`cds.test` / MockMvc) |
| **Review criteria** | Services are use-case projections, not 1:1 model dumps; declarative validation preferred over handler code; no accidental exposure |
| **Exit criteria** | API review PASS; API tests green; API docs current |

## M4 — Business Logic

| Aspect | Content |
|---|---|
| **Deliverables** | Custom event handlers (`before`/`on`/`after`; `@Before`/`@On`/`@After`) implementing the domain behavior that generic handlers don't cover; error handling per `CAP-ERR` |
| **Applicable standards** | `CAP-LOGIC`, `CAP-DB` (CQL-based data access), `CAP-TXN`, `CAP-ERR` |
| **Development activities** | Implement only what CAP doesn't provide generically; use CQL/CQN, not raw SQL; respect managed transactions; use `req.error`/`ServiceException` idioms |
| **Validation activities** | Self-check against `CAP-LOGIC`/`CAP-DB`/`CAP-TXN` rules; no framework-duplicating code |
| **Tests** | Behavior tests through service interfaces incl. unhappy paths (AI-TEST-001/-004); handler unit tests where warranted (Java) |
| **Review criteria** | Handlers in correct phases; no bypassed abstractions; errors localized and sanitized; logic testable and tested |
| **Exit criteria** | All acceptance criteria for in-scope behavior demonstrably tested; logic review PASS |

## M5 — Integration

| Aspect | Content |
|---|---|
| **Deliverables** | Remote-service consumption (imported APIs, destinations), event emission/consumption (messaging config), mocks for all remote dependencies |
| **Applicable standards** | `CAP-INT`, `CAP-EVT`, `CAP-ERR` (remote failure handling), `CAP-TEST` (mocked integration tests) |
| **Development activities** | Import external service models; use CAP remote-service and messaging abstractions (no hand-rolled HTTP/broker clients); design for remote failure |
| **Validation activities** | All external dependencies mockable and mocked locally; failure modes handled deliberately |
| **Tests** | Integration tests against mocks; contract assumptions documented; hybrid tests where cloud services are required |
| **Review criteria** | CAP-native consumption; resilience to remote failure; no credentials in code/config |
| **Exit criteria** | Integration review PASS; app fully runnable locally with mocks |

## M6 — Security

| Aspect | Content |
|---|---|
| **Deliverables** | Authorization model implemented (`@requires`/`@restrict`, instance-based restrictions), `xs-security.json`/IdP configuration, role collections documented; security notes for deviations |
| **Applicable standards** | `CAP-SEC`, `CAP-PRIV` (audit logging), `CAP-MT` (tenant isolation, if applicable) |
| **Development activities** | Apply deny-by-default authorization; map roles to scopes; wire production authentication (XSUAA/IAS); enable audit logging for personal-data access as applicable |
| **Validation activities** | Every exposed service/entity/action has a deliberate authorization decision; mocked auth confined to dev/test profiles |
| **Tests** | Authorization tests: allowed and denied access per role for every restricted resource (AI-TEST-004) |
| **Review criteria** | No unprotected exposure; no privilege escalation via unrestricted navigation/expand; secrets management clean |
| **Exit criteria** | Security review PASS with zero Critical/High findings; authorization test suite green |

## M7 — Testing (consolidation)

| Aspect | Content |
|---|---|
| **Deliverables** | Complete automated test suite (unit/service/integration per runtime guidance), test documentation, coverage report, reproducible test execution (one command) |
| **Applicable standards** | `CAP-TEST` (all), `AI-TEST` (all) |
| **Development activities** | Close coverage gaps found across M2–M6; stabilize flaky tests; ensure suite runs in CI |
| **Validation activities** | Coverage measured and honest gaps listed (AI-TEST-007); suite deterministic (AI-TEST-005) |
| **Tests** | The suite itself is the deliverable |
| **Review criteria** | Coverage of business behavior incl. unhappy paths; org coverage policy met ([research-gaps.md](../references/research-gaps.md) — SAP sets no threshold); no order-dependent tests |
| **Exit criteria** | Full suite green in CI; coverage accepted; known gaps documented |

## M8 — Deployment

| Aspect | Content |
|---|---|
| **Deliverables** | Deployment artifacts (mta.yaml or Helm charts/containers), CI/CD pipeline, environment configuration, successful deployment to a staging/test space, rollback approach |
| **Applicable standards** | `CAP-DEP`, `CAP-CICD`, `CAP-VER`, `CAP-SEC` (production auth active) |
| **Development activities** | `cds build`/MTA or Helm setup per target; bind platform services; externalize all configuration; pipeline with build+test+deploy stages |
| **Validation activities** | Clean deployment from pipeline (not laptop); no secrets in repo; dev-only features disabled in production profile |
| **Tests** | Pipeline-executed test stages; post-deployment smoke tests |
| **Review criteria** | Reproducible builds; environment parity dev→prod (database, auth); pipeline gates on test failure |
| **Exit criteria** | Green pipeline deploys to staging; deployment review PASS |

## M9 — Production Readiness

| Aspect | Content |
|---|---|
| **Deliverables** | Operations runbook, logging/monitoring/alerting configured (structured logs, telemetry, health checks), scaling configuration, data-privacy compliance evidence, final full-standard review report |
| **Applicable standards** | `CAP-OPS`, `CAP-LOG`, `CAP-PERF`, `CAP-PRIV`; plus re-check of all Critical/High rules across categories |
| **Development activities** | Wire observability (JSON logs, correlation IDs, telemetry exporters, health endpoints); load/perf validation against NFRs; finalize runbook |
| **Validation activities** | Operational drill: locate a request in logs end-to-end; health checks probed; alerts fire |
| **Tests** | Performance/load tests vs M0 NFRs; failover/restart behavior |
| **Review criteria** | The full [review model](../reviews/review-model.md) across all applicable categories; all Medium findings resolved or formally accepted |
| **Exit criteria** | Final review PASS; go-live approval recorded |
