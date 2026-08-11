# Architecture of the CAPM Engineering Standard

This document defines how the standard itself is built: its layers, its document model, its identifier scheme, and how the pieces support both **AI-assisted development** and **independent AI review**.

## Design goals

1. **Dual use.** The same rule base drives development (constructive: "build it this way") and review (evaluative: "check it was built this way"). Rules therefore carry both *good/bad examples* (for development) and *evidence + detection guidance* (for review).
2. **Traceable authority.** Every statement carries an [authority level](authority-levels.md) and, for SAP levels, a citation into the [source map](../references/sap-cap-sources.md).
3. **Version resilience.** CAP evolves continuously. Rules carry explicit CAP-version applicability and verification dates; version policy lives in [version-management.md](version-management.md).
4. **Modularity.** One rule = one addressable unit in a category file. Documents evolve independently; there is no monolithic standard document.
5. **Machine-followability.** The review and development procedures are written as deterministic instructions Claude Code can execute, not prose aspirations.

## The three layers

### Layer 1 — Engineering Principles
[`standards/principles/engineering-principles.md`](../standards/principles/engineering-principles.md)

A small set (~10) of fundamental principles — domain-first modeling, CAP-native-first, secure-by-design, testable business behavior, etc. Each principle is grounded in CAP documentation where possible and lists the Layer 2 rule categories that operationalize it. Principles are used to resolve conflicts between rules and to evaluate situations no rule covers.

### Layer 2 — Engineering Rules
[`standards/rules/`](../standards/rules/README.md)

The formal rule catalog. Rules describe **what a production-grade CAP application looks like**, independent of who builds it. Every rule follows [templates/rule-template.md](../templates/rule-template.md) with the mandatory fields:

Rule ID · Title · Category · Severity · Authority level · Applicability · Runtime applicability (Node.js/Java/Both) · CAP version applicability · Rule statement · Rationale · Evidence expected in code · Detection guidance · Good example · Bad example · Exception guidance · SAP reference · Last verified.

### Layer 3 — AI Development & Review Rules
[`standards/ai/`](../standards/ai/README.md)

Rules governing **Claude Code's own behavior**: how to inspect projects, design, implement, test, self-validate, review, report, handle uncertainty, and document decisions. These are `ORG` authority (they are our policy for AI usage), split into `AI-DEV-*`, `AI-REVIEW-*`, `AI-TEST-*`, `AI-DOC-*` families.

## Identifier scheme

```
<FAMILY>-<CATEGORY>-<NNN>
```

- Layer 2 family is `CAP`; Layer 3 families are `AI`.
- `NNN` is a zero-padded sequence per category, assigned once, **never reused or renumbered**. Retired rules are marked `Status: Deprecated` (with successor pointer) and remain in place.

### Layer 2 categories

| Prefix | Category | Rule file |
|---|---|---|
| `CAP-ARCH` | Architecture & project structure | `standards/rules/architecture.md` |
| `CAP-CDS` | Domain modeling & CDS | `standards/rules/cds-modeling.md` |
| `CAP-SRV` | Services & APIs | `standards/rules/services-apis.md` |
| `CAP-LOGIC` | Business logic & event handlers | `standards/rules/business-logic.md` |
| `CAP-DB` | Data access & persistence | `standards/rules/data-persistence.md` |
| `CAP-TXN` | Transactions | `standards/rules/transactions.md` |
| `CAP-INT` | Integration & remote services | `standards/rules/integration.md` |
| `CAP-EVT` | Events & messaging | `standards/rules/events-messaging.md` |
| `CAP-MT` | Multitenancy | `standards/rules/multitenancy.md` |
| `CAP-EXT` | Extensibility | `standards/rules/extensibility.md` |
| `CAP-SEC` | Security | `standards/rules/security.md` |
| `CAP-PRIV` | Data privacy & audit | `standards/rules/data-privacy.md` |
| `CAP-ERR` | Error handling | `standards/rules/error-handling.md` |
| `CAP-LOG` | Logging & observability | `standards/rules/logging-observability.md` |
| `CAP-TEST` | Testing | `standards/rules/testing.md` |
| `CAP-PERF` | Performance & scalability | `standards/rules/performance.md` |
| `CAP-DEP` | Deployment (CF, Kyma/K8s) | `standards/rules/deployment.md` |
| `CAP-CICD` | CI/CD | `standards/rules/cicd.md` |
| `CAP-VER` | Dependency & version management | `standards/rules/versions-dependencies.md` |
| `CAP-OPS` | Production readiness & operations | `standards/rules/production-readiness.md` |

### Severity scale

| Severity | Meaning in review | Meaning at a quality gate |
|---|---|---|
| **Critical** | Exploitable security flaw, data loss/corruption risk, broken tenant isolation, or production outage risk | Blocks the milestone unconditionally |
| **High** | Violates SAP requirement or core architecture; will cause defects, insecurity, or unmaintainability | Blocks the milestone unless a documented exception is approved |
| **Medium** | Violates recommendation; degrades quality, performance, or operability | Must be remediated or explicitly accepted before production readiness (M9) |
| **Low** | Improvement opportunity; style or minor deviation | Recorded; remediation optional |

## How the layers connect to the lifecycle

The [development lifecycle](../development/lifecycle.md) defines milestones M0–M9. Each milestone lists the Layer 2 categories in scope for its gate. The gate itself runs the [review model](../reviews/review-model.md) (Layer 3 `AI-REVIEW-*` behavior) over those categories:

```
DEVELOP → SELF-VALIDATE → TEST → CAPM STANDARD REVIEW → REMEDIATE → PASS MILESTONE
(Layer 3    (Layer 2 rules,  (CAP-TEST  (review-model.md,      (fix FAIL         (exit criteria
 AI-DEV)     self-check)      rules)     AI-REVIEW rules)       findings)          met)
```

## Document model

- `docs/` — the design of the standard (this file and its siblings). Changes rarely.
- `standards/` — the normative content. Changes deliberately, rule-by-rule.
- `development/`, `reviews/` — executable procedures for the two Claude Code use cases.
- `references/` — **evidence, not norms**: the SAP source map, gap register, and candidate-rule inventory. Keeping evidence separate from norms is what lets us re-verify against new CAP releases without rewriting rules.
- `templates/` — canonical formats for rules and reports.
