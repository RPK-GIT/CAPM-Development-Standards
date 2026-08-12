# Layer 2 — Engineering Rule Catalog

The formal rule catalog describing what a production-grade CAP application looks like. Every rule follows [templates/rule-template.md](../../templates/rule-template.md) and carries a stable ID, severity, [authority level](../../docs/authority-levels.md), runtime applicability, and CAP version applicability.

> **Status: Phase 2 rule authoring COMPLETE (2026-08-12).** All 20 categories are authored and normative — **134 rules**: `CAP-SEC` (18), `CAP-MT` (6), `CAP-ARCH` (7), `CAP-CDS` (11), `CAP-SRV` (9), `CAP-DB` (10), `CAP-TXN` (6), `CAP-EVT` (7), `CAP-LOGIC` (5), `CAP-INT` (7), `CAP-TEST` (7), `CAP-ERR` (6), `CAP-LOG` (5), `CAP-PERF` (7), `CAP-EXT` (4), `CAP-PRIV` (4), `CAP-DEP` (3), `CAP-CICD` (3), `CAP-VER` (6), `CAP-OPS` (3). Full disposition history: [candidate-dispositions.md](../../references/candidate-dispositions.md). The candidate inventory in [references/candidate-rules.md](../../references/candidate-rules.md) is now research history only.

## Categories

| Prefix | Category | File | Primary sources |
|---|---|---|---|
| `CAP-ARCH` | Architecture & project structure — **7 rules active** | [architecture.md](architecture.md) | CAP best practices, getting started |
| `CAP-CDS` | Domain modeling & CDS — **11 rules active** | [cds-modeling.md](cds-modeling.md) | Domain modeling guide, CDS reference |
| `CAP-SRV` | Services & APIs — **9 rules active** | [services-apis.md](services-apis.md) | Providing services guide, protocol adapters |
| `CAP-LOGIC` | Business logic & event handlers — **5 rules active** | [business-logic.md](business-logic.md) | Node.js/Java runtime docs |
| `CAP-DB` | Data access & persistence — **10 rules active** | [data-persistence.md](data-persistence.md) | Databases guide, CQL/CQN |
| `CAP-TXN` | Transactions — **6 rules active** | [transactions.md](transactions.md) | Node.js `cds.tx`, Java ChangeSet docs |
| `CAP-INT` | Integration & remote services — **7 rules active** | [integration.md](integration.md) | Consuming services guide |
| `CAP-EVT` | Events & messaging — **7 rules active** | [events-messaging.md](events-messaging.md) | Messaging guide |
| `CAP-MT` | Multitenancy — **6 rules active** | [multitenancy.md](multitenancy.md) | Multitenancy/MTX guide |
| `CAP-EXT` | Extensibility — **4 rules active** | [extensibility.md](extensibility.md) | Extensibility guide |
| `CAP-SEC` | Security — **18 rules active** | [security.md](security.md) | Security/authorization/authentication guides |
| `CAP-PRIV` | Data privacy & audit — **4 rules active** | [data-privacy.md](data-privacy.md) | Data privacy guide |
| `CAP-ERR` | Error handling — **6 rules active** | [error-handling.md](error-handling.md) | Runtime error-handling docs |
| `CAP-LOG` | Logging & observability — **5 rules active** | [logging-observability.md](logging-observability.md) | `cds.log`, telemetry docs |
| `CAP-TEST` | Testing — **7 rules active** | [testing.md](testing.md) | `cds.test`, CAP Java testing docs |
| `CAP-PERF` | Performance & scalability — **7 rules active** | [performance.md](performance.md) | Performance-relevant guides |
| `CAP-DEP` | Deployment (CF, Kyma/K8s) — **3 rules active** | [deployment.md](deployment.md) | Deployment guides |
| `CAP-CICD` | CI/CD — **3 rules active** | [cicd.md](cicd.md) | CI/CD guide |
| `CAP-VER` | Dependency & version management — **6 rules active** | [versions-dependencies.md](versions-dependencies.md) | Release schedule, version docs; live baseline: [version-management](../../docs/version-management.md) |
| `CAP-OPS` | Production readiness & operations — **3 rules active** | [production-readiness.md](production-readiness.md) | Deploy-to-production guidance |

## Catalog invariants

1. **Stable IDs.** `CAP-<CAT>-NNN` is assigned once and never reused or renumbered. Deprecated rules stay in the file with `Status: Deprecated` and a successor pointer.
2. **One rule, one concern.** A rule states one testable requirement. Compound expectations are split.
3. **Authority honesty.** A rule may only carry `SAP-REQ`/`SAP-REC` with a working official-SAP citation and a `Last verified` date. See [docs/authority-levels.md](../../docs/authority-levels.md).
4. **Runtime explicitness.** Every rule states Node.js / Java / Both. Runtime-specific mechanics (e.g., `cds.tx` vs `ChangeSetContext`) get runtime-specific rules that cross-link each other.
5. **Version sensitivity.** Rules affected by CAP releases carry explicit version boundaries; the register of version-sensitive rules is maintained in [docs/version-management.md](../../docs/version-management.md).
6. **Reviewability.** Every rule must be evaluable from repository contents alone (plus documented dynamic checks); if no objective detection guidance can be written, the rule is not ready for the catalog.
