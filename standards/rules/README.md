# Layer 2 — Engineering Rule Catalog

The formal rule catalog describing what a production-grade CAP application looks like. Every rule follows [templates/rule-template.md](../../templates/rule-template.md) and carries a stable ID, severity, [authority level](../../docs/authority-levels.md), runtime applicability, and CAP version applicability.

> **Status:** Phase 2 in progress. **Authored and normative: `CAP-SEC` (18 rules) and `CAP-MT` (6 rules)** — see [candidate-dispositions.md](../../references/candidate-dispositions.md) for how each candidate was disposed. All other categories remain candidate inventory in [references/candidate-rules.md](../../references/candidate-rules.md); candidates are *not normative* — do not cite them in reviews until promoted into a category file here.

## Categories

| Prefix | Category | File | Primary sources |
|---|---|---|---|
| `CAP-ARCH` | Architecture & project structure | [architecture.md](architecture.md) | CAP best practices, getting started |
| `CAP-CDS` | Domain modeling & CDS | [cds-modeling.md](cds-modeling.md) | Domain modeling guide, CDS reference |
| `CAP-SRV` | Services & APIs | [services-apis.md](services-apis.md) | Providing services guide, protocol adapters |
| `CAP-LOGIC` | Business logic & event handlers | [business-logic.md](business-logic.md) | Node.js/Java runtime docs |
| `CAP-DB` | Data access & persistence | [data-persistence.md](data-persistence.md) | Databases guide, CQL/CQN |
| `CAP-TXN` | Transactions | [transactions.md](transactions.md) | Node.js `cds.tx`, Java ChangeSet docs |
| `CAP-INT` | Integration & remote services | [integration.md](integration.md) | Consuming services guide |
| `CAP-EVT` | Events & messaging | [events-messaging.md](events-messaging.md) | Messaging guide |
| `CAP-MT` | Multitenancy — **6 rules active** | [multitenancy.md](multitenancy.md) | Multitenancy/MTX guide |
| `CAP-EXT` | Extensibility | [extensibility.md](extensibility.md) | Extensibility guide |
| `CAP-SEC` | Security — **18 rules active** | [security.md](security.md) | Security/authorization/authentication guides |
| `CAP-PRIV` | Data privacy & audit | [data-privacy.md](data-privacy.md) | Data privacy guide |
| `CAP-ERR` | Error handling | [error-handling.md](error-handling.md) | Runtime error-handling docs |
| `CAP-LOG` | Logging & observability | [logging-observability.md](logging-observability.md) | `cds.log`, telemetry docs |
| `CAP-TEST` | Testing | [testing.md](testing.md) | `cds.test`, CAP Java testing docs |
| `CAP-PERF` | Performance & scalability | [performance.md](performance.md) | Performance-relevant guides |
| `CAP-DEP` | Deployment (CF, Kyma/K8s) | [deployment.md](deployment.md) | Deployment guides |
| `CAP-CICD` | CI/CD | [cicd.md](cicd.md) | CI/CD guide |
| `CAP-VER` | Dependency & version management | [versions-dependencies.md](versions-dependencies.md) | Release schedule, version docs |
| `CAP-OPS` | Production readiness & operations | [production-readiness.md](production-readiness.md) | Deploy-to-production guidance |

## Catalog invariants

1. **Stable IDs.** `CAP-<CAT>-NNN` is assigned once and never reused or renumbered. Deprecated rules stay in the file with `Status: Deprecated` and a successor pointer.
2. **One rule, one concern.** A rule states one testable requirement. Compound expectations are split.
3. **Authority honesty.** A rule may only carry `SAP-REQ`/`SAP-REC` with a working official-SAP citation and a `Last verified` date. See [docs/authority-levels.md](../../docs/authority-levels.md).
4. **Runtime explicitness.** Every rule states Node.js / Java / Both. Runtime-specific mechanics (e.g., `cds.tx` vs `ChangeSetContext`) get runtime-specific rules that cross-link each other.
5. **Version sensitivity.** Rules affected by CAP releases carry explicit version boundaries; the register of version-sensitive rules is maintained in [docs/version-management.md](../../docs/version-management.md).
6. **Reviewability.** Every rule must be evaluable from repository contents alone (plus documented dynamic checks); if no objective detection guidance can be written, the rule is not ready for the catalog.
