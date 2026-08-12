# Research Gaps — Where SAP Guidance Ends and Organization Policy Must Begin

Register of areas where official SAP CAP documentation (as verified 2026-08-11, see [sap-cap-sources.md](sap-cap-sources.md)) gives **no explicit guidance**, gives only mechanisms without policy, or explicitly delegates responsibility to the application/organization. Each gap needs an **ORG** decision before the corresponding rules can be Critical/High gate criteria.

Legend — SAP position: **explicit-gap** = SAP states it's the app's responsibility · **mechanism-only** = SAP documents the how, not the policy · **silent** = no guidance found · **in-flux** = SAP documentation incomplete/unstable.

| # | Area | Gap | SAP position | Proposed owner / next step |
|---|---|---|---|---|
| G-01 | Testing | Code-coverage thresholds, test-pyramid ratios, mandatory test layers, CI gating criteria | silent | ORG policy still open — deliberately NOT manufactured in the authored CAP-TEST rules; behavior-verification duties are CAP-TEST-007 and the per-rule test evidence |
| G-02 | Error handling | Application error-code taxonomy/naming (SAP shows i18n mechanics only) | mechanism-only | ORG error-code standard still open. *Stable-code duty (scheme-agnostic) now CAP-ERR-001/-005* |
| G-03 | Logging | What business data may be logged, retention periods, audit-vs-application log boundary | mechanism-only (masking defaults exist) | ORG + legal still open. *Content hygiene is CAP-SEC-016; CAP-LOG deliberately does not equate application and audit logs (audit mechanism: CAP-PRIV-002)* |
| G-04 | Performance | Budgets/SLOs, load-test methodology, sizing (pools, instances), concrete pagination values | mechanism-only | ORG NFR standard per project (M0) still open — CAP-PERF deliberately defines no thresholds; anti-pattern rules are CAP-PERF-001..007 |
| G-05 | Performance | N+1 in custom handler code (no explicit SAP rule; `$expand` primitives documented) | mechanism-only | **Resolved:** authored as CAP-PERF-007 (GEN authority — honest non-SAP labeling); remote fan-out is CAP-INT-005 |
| G-06 | Security | Authorization audit trail — CAP does **not** log authz successes/failures automatically | explicit-gap | ORG custom-handler pattern needed |
| G-07 | Security | Rate limiting: named a developer responsibility, no thresholds/implementation | explicit-gap | ORG limits + implementation choice (app vs route service). *Decision enforced (not the values) by CAP-SEC-014* |
| G-08 | Security | Node.js `$expand` depth limiting requires custom handler — no reference implementation | explicit-gap | ORG reference handler + max depth. *Decision enforced by CAP-SEC-014* |
| G-09 | Security | Content Security Policy required ("applications have to configure CSP") but no baseline given | explicit-gap | ORG CSP baseline |
| G-10 | Security | IAS-without-AMS authorization mapping not standardized | explicit-gap | ORG: mandate AMS (aligned with SAP IAS-first direction) |
| G-11 | Security | Malware scanning for uploads: custom handlers required, no reference pattern | explicit-gap | ORG pattern for file-upload endpoints |
| G-12 | Security | Java composition-child authorization gap: documented, no standard mitigation | explicit-gap | Review duty now formalized as CAP-SEC-011; a reusable ORG mitigation pattern (reference handler) still open |
| G-13 | Security | Secret rotation cadence ("rotate regularly", no interval) | mechanism-only | ORG rotation periods |
| G-14 | Security | Role-collection governance / least-privilege review process | silent | ORG governance with BTP admins |
| G-15 | Security | `model-strict` (Java) not mandated; no Node equivalent lint | mechanism-only | ORG: decide to require it + CI check. *CAP-SEC-004 forbids weakening defaults and records `model-strict` adoption as an observation pending this decision* |
| G-16 | Privacy | Legal sufficiency of DPP features explicitly disclaimed by SAP; retention periods, legal grounds | explicit-gap | Legal/compliance per jurisdiction — still open. *The documented-approach duty (not the periods) is now CAP-PRIV-004 (GEN)* |
| G-17 | Privacy | Data Retention Manager guide **under construction** (re-verified 2026-08-12: content unrendered); `@cap-js/data-privacy` (DPI) is the apparent forward path | in-flux | Watch SAP docs; interim approach mandated by CAP-PRIV-004 without SAP authority |
| G-18 | Modeling | CDS file organization at scale (multi-file layout, index.cds conventions) | silent | ORG convention still open. *Baseline layout/conventions now CAP-ARCH-001; concern separation CAP-CDS-007* |
| G-19 | Modeling | Naming vocabulary beyond casing (services, actions, events, multi-word style) | mechanism-only (examples only) | ORG naming standard still open. *SAP's documented checklist is now CAP-CDS-001* |
| G-20 | Modeling | Enum vs CodeList decision rule; composition depth limits; draft-enablement criteria | silent | ORG modeling guidance still open. *Draft mechanics governed by CAP-SRV-007; containment semantics by CAP-CDS-006* |
| G-21 | Architecture | Microservice-cut criteria ("late-cut" advised, no thresholds) | mechanism-only | ORG criteria still open. *Modulith default + ADR-per-cut now enforced by CAP-ARCH-006/-007* |
| G-22 | Architecture | API/service versioning, breaking-change and deprecation management | silent | ORG API lifecycle policy |
| G-23 | Architecture | Multi-protocol exposure policy (same service via OData+REST+GraphQL); GraphQL adapter support stance (plugin, not core) | silent | ORG protocol governance still open. *Exposure explicitness now enforced by CAP-SRV-006* |
| G-24 | Data | Raw-SQL governance in Java (`JdbcTemplate` permitted, no criteria/review rules) | mechanism-only | ORG criteria still open. *Isolation/documentation duty now CAP-DB-004* |
| G-25 | Data | PostgreSQL-in-production criteria ("edge cases" undefined; no MT/extensibility support) | mechanism-only | ORG decision matrix still open. *ADR duty per deviation now CAP-DB-001* |
| G-26 | Data | Test-data vs production seed-data separation policy (beyond `cds deploy --production` semantics) | mechanism-only | ORG data policy |
| G-27 | Integration | Node.js resilience for remote calls (timeouts/retries/circuit breakers — Java gets ResilienceDecorator, Node only community/mesh pointers) | mechanism-only | ORG resilience standard (mechanism + thresholds) still open. *The deliberate-failure-design duty is now CAP-INT-007* |
| G-28 | Integration | Destination governance (naming, per-environment segregation, credential rotation) | silent | ORG destination standard still open. *MT retrieval-strategy decisions now CAP-INT-006; credential hygiene CAP-SEC-017* |
| G-29 | Integration | Live remote access vs replication/federation thresholds | mechanism-only | ORG criteria still open. *Mashup/expand bounding now CAP-INT-005* |
| G-30 | Events | Domain-event naming/namespacing/versioning scheme for own events | mechanism-only (examples only) | ORG event-design standard still open. *Modeled-event requirement now CAP-EVT-001* |
| G-31 | Events | Broker selection matrix (Event Hub vs Event Mesh vs non-SAP brokers) | mechanism-only ("new default" statement only) | ORG selection policy still open. *Default + documented-reason duty now CAP-EVT-006* |
| G-32 | Events | Dead-letter operations: ownership, alerting thresholds, SLAs ("manual intervention required") | explicit-gap | ORG runbook (feeds M9) still open. *The operational-process requirement itself is now CAP-EVT-005* |
| G-33 | Events | Long-running/background-work idempotency, retry, compensation patterns (no distributed atomicity) | mechanism-only | ORG compensation-pattern standard still open. *Invariants now enforced: CAP-TXN-005 (no atomicity assumptions), CAP-EVT-003 (idempotency)* |
| G-34 | Multitenancy | Tenant-upgrade orchestration: zero-downtime, batching, canary tenants, rollback | mechanism-only | ORG upgrade runbook. *The upgrade-before-serve invariant itself is now CAP-MT-005* |
| G-35 | Extensibility | Extension-allowlist content (field caps, namespaces, annotation whitelist) | explicit-gap (mechanism default-forbidden) | ORG per-product allowlist still open. *The configured-allowlist duty is now CAP-EXT-002* |
| G-36 | Operations | Consolidated production-readiness / go-live checklist (SAP has none; prep scattered in to-cf guide) | silent | **Resolved:** CAP-OPS-003 (ORG) binds the M9 gate as a formal rule with defined pass criteria and evidence |
| G-37 | Operations | Alerting thresholds, probe intervals, CF readiness-check tooling gap workaround | mechanism-only | ORG ops standard still open. *Telemetry mechanism-when-adopted is CAP-LOG-004; no monitoring mandates invented* |
| G-38 | Operations | Container/image policy for Kyma (scanning, signing, base-image updates) | silent | ORG container policy |
| G-39 | CI/CD | Branch policies, scanning depth, artifact retention, blue-green/canary strategies | mechanism-only (scaffolding only) | ORG pipeline specifics still open. *The minimum gate set and enforcement-class model are now CAP-CICD-003 (ORG)* |
| G-40 | Versions | Dependency-update SLA (patch within N days); no concrete EOL dates published per CAP version | mechanism-only | ORG SLA still open. *Freeze+refresh duty is CAP-VER-001; line currency is CAP-VER-002* |
| G-41 | AI/MCP | MCP adapter governance: SAP documents missing protections (injection, rate limits, audit) with no timeline; `@cap-js/mcp-server` has no stability label or security guidance | explicit-gap / in-flux | ORG AI-exposure policy before any use beyond local dev. *Governance requirement formalized as CAP-SEC-018; the concrete compensating-controls policy is still this open ORG decision* |
| G-42 | Docs stability | capire URL churn (Jan 2026 restructure; several pages 404/unreleased/under construction) | in-flux | Quarterly re-verification per [version-management](../docs/version-management.md) |
| G-43 | Security | XSUAA→IAS migration policy for **existing** applications: SAP recommends IAS for new projects and provides cross-consumption as a bridge, but mandates no migration or timeline for existing XSUAA apps | mechanism-only | ORG migration policy (whether/when to migrate, per-app criteria). Referenced by CAP-SEC-006, which covers new projects only |

## How gaps are closed

1. A gap owner drafts an **ORG** rule (or accepts the risk explicitly) using the [rule template](../templates/rule-template.md).
2. The rule enters the Layer 2 catalog with authority `ORG` and a link back to its gap entry.
3. The gap row is updated with the rule ID; the row stays (as history) until SAP publishes guidance, at which point the rule is re-based on the SAP source and re-authorized.
