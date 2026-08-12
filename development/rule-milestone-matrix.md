# Rule–Milestone Matrix

The canonical mapping between the [Layer 2 rule catalog](../standards/rules/README.md) (134 rules) and the [M0–M9 lifecycle](lifecycle.md). This document defines **when and how each rule is enforced** — the catalog remains the authoritative definition of *what* each rule requires. Per-milestone operational checklists: [milestones/](milestones/m0-requirements.md).

Verified against the catalog on **2026-08-12** (all 134 rules mapped; validation results at the end).

---

## 1. Classification models

### 1.1 Applicability (per rule × milestone)

| Class | Meaning |
|---|---|
| **PRIMARY** | The milestone where the rule is mainly evaluated — evidence is produced and assessed here first |
| **SUPPORTING** | Considered at this milestone; primarily owned elsewhere. Findings are reported (security rules additionally escalate via the report's cross-cutting security observation mechanism) but do **not** by themselves gate this milestone — the gate class applies where the rule is PRIMARY or FINAL-GATE *(clarified 2026-08-12, Round 1 calibration)* |
| **FINAL-GATE** | Must be explicitly re-verified before this milestone can PASS (typically after later work could have invalidated earlier verification) |
| **CONDITIONAL** | Applies only when the project uses the relevant capability (see §1.4). Conditions combine with the other classes — a rule can be CONDITIONAL *and* PRIMARY |

### 1.2 Gate class (per rule — applied where the rule is evaluated as PRIMARY or FINAL-GATE; SUPPORTING appearances are non-gating, see §1.1)

| Class | Meaning | Passage on violation |
|---|---|---|
| **HARD-GATE** | Genuinely consequential: security, tenant isolation, severe data integrity, injection, production compatibility, critical-security testing, production configuration | Milestone cannot PASS without an approved exception ([AI-DOC-002](../standards/ai/ai-documentation-rules.md) record: rule ID, reason, scope, approver, expiry/review condition) |
| **SOFT-GATE** | Must be reported and normally remediated | May proceed with explicit, recorded justification in the milestone checklist |
| **ADVISORY** | Guidance to consider | Recorded; never blocks |

Gate class is **not** derived mechanically from authority or severity: e.g., CAP-SEC-004 is SAP-REC yet HARD (weakened authentication surface); CAP-VER-001 is SAP-REQ-worded yet SOFT (reproducibility drift, exception-friendly); CAP-TXN-005 is High/SAP-REQ yet SOFT (its own statement builds in a documented-acceptance path). Unusual classifications are footnoted in §2.

### 1.3 Milestone gate results

`PASS` · `PASS WITH EXCEPTIONS` (all HARD-GATE violations covered by approved AI-DOC-002 exceptions; soft-gate justifications recorded) · `FAIL` (≥ 1 uncovered HARD-GATE violation) · `NOT READY` (required evidence missing — maps to per-rule NOT ASSESSABLE at scale) · `NOT APPLICABLE` (milestone/capability out of scope for this project). These refine the lifecycle's severity gating (Critical always blocks; High blocks unless excepted; Medium resolved-or-accepted by M9; Low recorded) — the two models are consistent: every Critical rule is HARD; High rules are HARD or SOFT per §1.2's criteria.

### 1.4 Capability profile (drives CONDITIONAL rules)

Determined at **M0** and kept in the project profile (review-model step 2). Flags:

`MT` multitenant · `EVENTING` async events/queued processing · `BROKER` message broker · `REMOTE` remote-service consumption · `MASHUP` cross local/remote reads · `ODATA` OData-served APIs · `CUSTOM-OPS` actions/functions · `DRAFTS` draft editing · `MEDIA` media/large binaries · `LOCALIZED` translatable data · `TEMPORAL` time-sliced data · `PERSONAL-DATA` personal data processed · `PDM` Personal Data Manager · `EXTENSIBILITY` SaaS customer extensions offered · `FEATURE-TOGGLES` fts/ features · `MCP` `@mcp`-exposed services · `IAS` / `XSUAA` identity service · `NEW-PROJECT` greenfield · `CF` / `KYMA` deployment target · `MULTI-LANDSCAPE` >1 environment · `HANA-LARGE` large-volume HANA entities · `PESSIMISTIC-LOCKING` · `CONCURRENT-EDIT` concurrently edited entities · `CONCURRENT-PAGING` paged-while-modified collections · `MASS-DATA` batch/export paths · `ERROR-HOOKS` custom `srv.on('error')` · `UI` end-user UI · `TELEMETRY` tracing/metrics adopted · `CLOUD-ONLY` parity-sensitive features (locking/MTX/HANA-specific) · `MAJOR-UPGRADE` CAP major upgrade in scope · `LEGACY-HDBCDS` migrated on-prem-origin HANA artifacts · `NODE` / `JAVA` runtime (from the profile; runtime-scoped rules are implicitly conditional on it).

### 1.5 Evidence quality (used in reviews)

`DIRECT` (demonstrates compliance) · `INDIRECT` (supports, not definitive) · `MISSING` (expected, absent) · `CONTRADICTORY` (indicates violation) · `NOT-ASSESSABLE` (repository lacks sufficient evidence — feeds NOT READY). Each rule's **Evidence expected in code** + **Detection guidance** sections define what DIRECT evidence looks like; milestone checklists list the artifact classes to collect (models, service definitions, handlers, tests, manifests, descriptors, pipeline config, ADRs, command/test output).

### 1.6 ORG and GEN rules

- **ORG rules (3) — status `ORG-PENDING` until organizational ratification:** CAP-ARCH-007, CAP-CICD-003, CAP-OPS-003. They keep their designed gate class below, but every finding against them MUST be reported as *"ORG-PENDING policy finding — blocking authority derives from this standard's own governance, not SAP; subject to ratification"*. They MUST NOT be presented as SAP requirements, and compliance with them MUST NOT be reported as SAP compliance (AI-DOC-004). After ratification the organization may adjust their gate behavior; their authority stays ORG.
- **GEN rules (2):** CAP-PERF-007 (SOFT at M4 — real resource risk, but general practice with legitimate bounded-loop exceptions) and CAP-PRIV-004 (SOFT at M9 — compliance-engineering duty; periods/legal grounds are outside the standard). Reported as general engineering guidance, never as SAP requirements.

---

## 2. The matrix

Legend: **Gate** H = HARD, S = SOFT, A = ADVISORY. **RT** B/N/J. **Cond** = capability condition (— = unconditional). Milestone columns: P = PRIMARY, Su = SUPPORTING, FG = FINAL-GATE. Severity/authority are as defined in the catalog (not repeated per row — Sev/Auth shown for orientation).

| Rule | Sev | Auth | RT | Gate | Cond | Primary | Supporting | Final-gate |
|---|---|---|---|---|---|---|---|---|
| CAP-ARCH-001 | M | REC | B | S | — | M1 | M2 | — |
| CAP-ARCH-002 | H | REC | B | S | — | M1 | M4 | — |
| CAP-ARCH-003 | H | REC | B | S | — | M1 | M3 | — |
| CAP-ARCH-004 | H | REC | B | S | — | M1 | M4 | — |
| CAP-ARCH-005 | M | REC | B | S | — | M1 | M4 | — |
| CAP-ARCH-006 | M | REC | B | S | — | M1 | — | — |
| CAP-ARCH-007 | M | ORG† | B | S | — | M1 | — | M9 |
| CAP-CDS-001 | L | REC | B | A | — | M2 | — | — |
| CAP-CDS-002 | M | REC | B | S | — | M2 | — | — |
| CAP-CDS-003 | M | REC | B | S | — | M2 | M5 | — |
| CAP-CDS-004 | M | REC | B | S | — | M2 | — | — |
| CAP-CDS-005 | M | REC | B | S | — | M2 | — | — |
| CAP-CDS-006 | H | REC | B | S | — | M2 | — | — |
| CAP-CDS-007 | M | REC | B | S | — | M2 | M3 | — |
| CAP-CDS-008 | M | REC | B | S | LOCALIZED | M2 | — | — |
| CAP-CDS-009 | M | REQ | B | S | TEMPORAL | M2 | M4 | — |
| CAP-CDS-010 | L | REC | B | A | — | M2 | — | — |
| CAP-CDS-011 | M | REC | B | S | — | M2 | — | — |
| CAP-SRV-001 | H | REC | B | S | — | M3 | M6 | — |
| CAP-SRV-002 | H | REC | B | S | — | M3 | M4 | — |
| CAP-SRV-003 | M | REC | B | S | — | M3 | M4 | — |
| CAP-SRV-004 | M | REQ | B | S | CUSTOM-OPS | M3 | M4 | — |
| CAP-SRV-005 | H | REQ | B | S | ODATA | M3 | — | — |
| CAP-SRV-006 | M | REC | B | S | — | M3 | M6 | — |
| CAP-SRV-007 | M | REC | B | S | DRAFTS | M3 | — | — |
| CAP-SRV-008 | M | REC | B | S | CONCURRENT-EDIT | M3 | — | — |
| CAP-SRV-009 | M | REC | B | S | MEDIA | M3 | — | — |
| CAP-LOGIC-001 | M | REC | B | S | — | M4 | — | — |
| CAP-LOGIC-002 | M | REC | B | S | — | M4 | — | — |
| CAP-LOGIC-003 | M | REQ | N | S | — | M4 | — | — |
| CAP-LOGIC-004 | M | REC | J | S | — | M4 | — | — |
| CAP-LOGIC-005 | M | REC | J | S | — | M4 | — | — |
| CAP-DB-001 | H | REC | B | S | — | M1 | — | M8 |
| CAP-DB-002 | H | REQ | B | **H** | — | M8 | M7 | — |
| CAP-DB-003 | H | REC | N | S | — | M1 | — | M8 |
| CAP-DB-004 | H | REC | B | S | — | M4 | — | — |
| CAP-DB-005 | M | REQ | N | S | — | M4 | — | — |
| CAP-DB-006 | M | REQ | N | S | LOCALIZED | M4 | — | — |
| CAP-DB-007 | M | REC | B | S | HANA-LARGE | M2 | M8 | — |
| CAP-DB-008 | M | REC | B | S | PESSIMISTIC-LOCKING | M4 | M7 | — |
| CAP-DB-009 | M | REC | N | S | — | M4 | — | — |
| CAP-DB-010 | L | REC | J | A | — | M4 | — | — |
| CAP-TXN-001 | H | REC | B | S | — | M4 | — | — |
| CAP-TXN-002 | M | REC | N | S | — | M4 | — | — |
| CAP-TXN-003 | L | REC | N | A | — | M4 | — | — |
| CAP-TXN-004 | M | REC | J | S | — | M4 | — | — |
| CAP-TXN-005 | H | REQ | B | S¹ | — | M4 | M5 | — |
| CAP-TXN-006 | H | REQ | N | **H** | — | M4 | M7 | — |
| CAP-INT-001 | H | REQ | B | **H** | REMOTE | M5 | — | — |
| CAP-INT-002 | H | REC | B | S | REMOTE | M5 | M1 | — |
| CAP-INT-003 | M | REC | B | S | REMOTE | M5 | — | — |
| CAP-INT-004 | M | REC | B | S | REMOTE | M5 | M7 | — |
| CAP-INT-005 | H | REC | B | S | MASHUP | M5 | — | — |
| CAP-INT-006 | M | REC | B | S | REMOTE+MT | M5 | M8 | — |
| CAP-INT-007 | M | REC | B | S | REMOTE | M5 | M9 | — |
| CAP-EVT-001 | M | REC | B | S | EVENTING | M5 | — | — |
| CAP-EVT-002 | C | REC | B | **H** | EVENTING | M5 | — | M8, M9 |
| CAP-EVT-003 | C | REQ | B | **H** | EVENTING | M5 | M7 | M9 |
| CAP-EVT-004 | H | REQ | B | **H** | EVENTING | M5 | M6 | — |
| CAP-EVT-005 | H | REC | B | S | EVENTING | M5 | — | M9 |
| CAP-EVT-006 | M | REC | B | S | EVENTING+BROKER | M1 | M5 | — |
| CAP-EVT-007 | L | REC | B | A | EVENTING | M5 | — | — |
| CAP-SEC-001 | C | REQ | B | **H** | — | M3 | — | M6, M9 |
| CAP-SEC-002 | C | REQ | B | **H** | — | M6 | — | M8, M9 |
| CAP-SEC-003 | C | REQ | N | **H** | — | M6 | — | M8 |
| CAP-SEC-004 | H | REC | J | **H**² | — | M6 | — | M8 |
| CAP-SEC-005 | C | REQ | J | **H** | — | M6 | — | M8 |
| CAP-SEC-006 | M | REC | B | S | NEW-PROJECT | M1 | — | — |
| CAP-SEC-007 | H | REQ | B | **H** | IAS | M6 | M3 | — |
| CAP-SEC-008 | H | REQ | B | **H** | XSUAA | M6 | M8 | — |
| CAP-SEC-009 | C | REQ | B | **H** | — | M6 | — | M9 |
| CAP-SEC-010 | H | REC | B | S | INSTANCE-AUTH | M3 | M4 | M6 |
| CAP-SEC-011 | H | REQ | B | **H** | — | M3 | — | M6 |
| CAP-SEC-012 | H | REC | B | S | — | M3 | M4 | M6 |
| CAP-SEC-013 | C | REQ | B | **H** | — | M4 | M3³ | M6 |
| CAP-SEC-014 | M | REQ | B | S | — | M6 | M3 | — |
| CAP-SEC-015 | C | REQ | B | **H** | — | M6 | — | M8, M9 |
| CAP-SEC-016 | M | REC | B | S | — | M4 | — | M6 |
| CAP-SEC-017 | H | REC | B | **H**² | — | M6 | M5 | M8 |
| CAP-SEC-018 | H | REC | B | **H**² | MCP | M3 | — | M6 |
| CAP-MT-001 | H | REQ | B | **H** | MT | M1 | M8 | — |
| CAP-MT-002 | H | REQ/REC | B | S | MT | M1 | — | M8 |
| CAP-MT-003 | C | REQ | B | **H** | MT | M6 | M4 | M9 |
| CAP-MT-004 | H | REQ | B | S | MT | M6 | M5 | — |
| CAP-MT-005 | C | REQ | B | **H** | MT | M8 | — | M9 |
| CAP-MT-006 | H | REC | B | S | MT | M4 | M5 | — |
| CAP-TEST-001 | M | REQ/REC | N | S | — | M7 | M4 | — |
| CAP-TEST-002 | M | REC | J | S | — | M7 | — | — |
| CAP-TEST-003 | M | REC | N | S | — | M7 | — | — |
| CAP-TEST-004 | L | REC | N | A | — | M7 | — | — |
| CAP-TEST-005 | M | REC | B | S | — | M7 | M6 | — |
| CAP-TEST-006 | M | REC | B | S | CLOUD-ONLY | M7 | M8 | — |
| CAP-TEST-007 | H | REC | B | **H**² | — | M7 | M6 | M9 |
| CAP-ERR-001 | M | REC | B | S | — | M4 | M3 | — |
| CAP-ERR-002 | H | REQ | N | **H** | — | M4 | — | M6 |
| CAP-ERR-003 | H | REC | B | S | — | M4 | — | — |
| CAP-ERR-004 | M | REQ | N | S | ERROR-HOOKS | M4 | — | — |
| CAP-ERR-005 | M | REC | B | S | UI | M4 | —⁴ | — |
| CAP-ERR-006 | H | REQ | B | **H** | — | M4 | — | M6 |
| CAP-LOG-001 | M | REC | B | S | — | M4 | — | — |
| CAP-LOG-002 | M | REC | B | S | — | M8 | — | M9 |
| CAP-LOG-003 | M | REC | B | S | — | M9 | M5 | — |
| CAP-LOG-004 | M | REC | B | S | TELEMETRY | M9 | — | — |
| CAP-LOG-005 | H | REC | J | **H**² | — | M8 | — | M9 |
| CAP-PERF-001 | M | REC | B | S | CONCURRENT-PAGING+ODATA | M3 | — | — |
| CAP-PERF-002 | M | REC | B | S | — | M2 | — | — |
| CAP-PERF-003 | H | REC | B | S | — | M2 | M4 | — |
| CAP-PERF-004 | M | REC | B | S | — | M2 | — | — |
| CAP-PERF-005 | M | REC | N | S | MASS-DATA | M4 | — | — |
| CAP-PERF-006 | M | REC | B | S | DRAFTS | M2 | — | — |
| CAP-PERF-007 | H | GEN† | B | S | — | M4 | — | — |
| CAP-EXT-001 | M | REC | B | S | — | M2 | M1 | — |
| CAP-EXT-002 | H | REQ | B | **H** | EXTENSIBILITY | M5 | M6 | — |
| CAP-EXT-003 | M | REC | B | S | EXTENSIBILITY | M5 | — | — |
| CAP-EXT-004 | M | REQ | B | S | FEATURE-TOGGLES | M5 | M8 | — |
| CAP-PRIV-001 | H | REC | B | **H**² | PERSONAL-DATA | M2 | — | M6, M9 |
| CAP-PRIV-002 | H | REC | B | **H** | PERSONAL-DATA | M6 | M8 | M9 |
| CAP-PRIV-003 | H | REQ | B | **H** | PDM | M6 | — | — |
| CAP-PRIV-004 | M | GEN† | B | S | PERSONAL-DATA | M9 | M6 | — |
| CAP-DEP-001 | M | REC | B | S | CF | M8 | — | — |
| CAP-DEP-002 | M | REC | B | S | MULTI-LANDSCAPE | M8 | — | — |
| CAP-DEP-003 | M | REC | B | S | KYMA | M8 | — | — |
| CAP-CICD-001 | H | REC | B | S | — | M8 | M7 | — |
| CAP-CICD-002 | M | REC | B | S | — | M8 | — | — |
| CAP-CICD-003 | M | ORG† | B | S | — | M8 | — | M9 |
| CAP-VER-001 | H | REQ | N | S¹ | — | M8 | — | — |
| CAP-VER-002 | H | REQ | B | **H** | — | M1 | — | M8, M9 |
| CAP-VER-003 | H | REQ | B | **H** | — | M1 | — | M8 |
| CAP-VER-004 | H | REQ | N | **H** | — | M8 | M1 | — |
| CAP-VER-005 | M | REC | B | S | MAJOR-UPGRADE | M8 | — | — |
| CAP-VER-006 | M | REQ | B | S | LEGACY-HDBCDS | M8 | M2 | — |
| CAP-OPS-001 | M | REC | B | S | — | M8 | — | M9 |
| CAP-OPS-002 | M | REC | B | S | UI | M9 | M8 | — |
| CAP-OPS-003 | H | ORG† | B | **H** | — | M9 | — | — |

† ORG rules are `ORG-PENDING` (§1.6); GEN rules are general practice — both reported per §1.6, never as SAP requirements.

**Footnoted classifications (deviations from a naive severity/authority mapping):**
¹ CAP-TXN-005 and CAP-VER-001 are SAP-REQ-worded but SOFT: TXN-005's own statement provides a documented-acceptance path for partial-failure risk; VER-001's failure mode is reproducibility drift with a routine remediation (regenerate lockfile) — blocking is disproportionate.
² HARD despite SAP-REC authority: CAP-SEC-004 (weakened authentication surface), CAP-SEC-017 (secrets exposure — a named critical candidate), CAP-SEC-018 (ungoverned MCP exposure incl. the documented SAP-API prohibition), CAP-LOG-005 (public actuators = credential-grade disclosure), CAP-TEST-007 (mandatory testing of critical security behavior — named hard-gate candidate), CAP-PRIV-001 (unclassified personal data disables every downstream protection). Gate class follows consequence, not wording strength.
³ CAP-SEC-013's M3 SUPPORTING appearance is conditional on `CUSTOM-OPS ∧ (REMOTE ∨ MASHUP)` (custom operations exist and remote services/mashups are consumed — the surface where handler-built queries/URLs appear before M4). Added 2026-08-12 (Round 1 calibration, Pilot 1 evidence: a genuine injection pattern was reviewable at M3 but out of scope). Non-gating at M3 per §1.1; the HARD gate applies at its PRIMARY milestone M4 and FINAL-GATE M6.
⁴ CAP-ERR-005's former M3 SUPPORTING appearance was removed 2026-08-12 (Round 1 calibration): Pilot 1 showed the finding is valid but not actionable until M4 handler work — the M3 error-contract sketch is CAP-ERR-001's subject, which remains M3 SUPPORTING. No severity/authority change.

---

## 3. Critical-rule coverage report

All **11 Critical rules**; none unmapped. Exception mechanism for all: AI-DOC-002 record (rule ID, deviation, impact, compensating controls, approver, expiry/review) — Critical exceptions additionally require the risk acceptance to name the catalog's "Critical justification" being accepted.

| Rule | Title (short) | Cond | Primary | Final-gate(s) | Gate | Blocks? | Required evidence (DIRECT) |
|---|---|---|---|---|---|---|---|
| CAP-SEC-001 | Explicit authorization on every service | — | M3 | M6, M9 | HARD | Yes | `@requires`/`@restrict` per exposed service (srv/*.cds + annotate files); allow/deny tests |
| CAP-SEC-002 | Real identity service in production | — | M6 | M8, M9 | HARD | Yes | Production auth kind (config profiles); xsuaa/identity resource in mta.yaml/chart |
| CAP-SEC-003 | Node deny-by-default preserved | NODE | M6 | M8 | HARD | Yes | Absence of `restrict_all_services: false` in all config |
| CAP-SEC-005 | Java security activation verified | JAVA | M6 | M8 | HARD | Yes | `cds-feature-identity` in pom + identity binding + unauthenticated-rejection check |
| CAP-SEC-009 | Technical roles never business-assigned | — | M6 | M9 | HARD | Yes | xs-security.json role-template scopes; app-to-app grants; cockpit export if needed |
| CAP-SEC-013 | Injection-safe queries | — | M4 | M6 | HARD | Yes | Handler scan (no string-built queries; params; positive lists) with file:line |
| CAP-SEC-015 | Backend auth independent of App Router | — | M6 | M8, M9 | HARD | Yes | Pipeline/post-deploy 401/403 smoke check; backend auth config |
| CAP-MT-003 | Strict tenant isolation | MT | M6 | M9 | HARD | Yes | No static/module-level tenant-unsafe state (scan); isolation tests (t1/t2) |
| CAP-MT-005 | Tenants upgraded before serving | MT | M8 | M9 | HARD | Yes | Upgrade step in deployment automation, ordered before traffic |
| CAP-EVT-002 | Transactional queue on in production | EVENTING | M5 | M8, M9 | HARD | Yes | No unqueued/disabled-outbox production config; justified `cds.unqueued` sites |
| CAP-EVT-003 | Idempotent event handlers | EVENTING | M5 | M9 | HARD | Yes | Handler idempotency mechanisms with file:line; duplicate-delivery tests |

Every Critical rule is HARD, blocks progression at its primary and final-gate milestones, and is visible at M9 either as an explicit FINAL-GATE or through CAP-OPS-003's full-review aggregation.

---

## 4. Review-efficiency model

A milestone review loads: **(1)** the milestone's PRIMARY + FINAL-GATE rules, **(2)** its SUPPORTING rules *only where the milestone's changes touch their subject*, **(3)** CONDITIONAL rules filtered by the M0 capability profile and runtime. Nothing else is evaluated. The largest single-milestone load is M4 (29 primary); a Node-only, single-tenant, no-eventing project's M4 drops to ~21 rules after filtering. Full-catalog evaluation happens exactly once: the M9 review (per CAP-OPS-003), scoped by profile.

**Security stays visible across milestones by design:** SEC rules are PRIMARY at M3/M4/M6 and FINAL-GATE at M6/M8/M9 — passing M6 does not immunize later changes; M8/M9 re-verify the production-configuration security gates (SEC-002/-003/-004/-005/-015/-017), and M9 re-verifies exposure/tenancy (SEC-001/-009, MT-003).

**Version-sensitive rules** (⏱ in the catalog) bind the [version register](../docs/version-management.md) — checklists never hard-code versions; reviewers first confirm the register's verification date is current.

---

## 5. Validation summary (2026-08-12)

- **134/134 rules mapped**; every rule has ≥ 1 PRIMARY milestone, a gate class, and an evidence path (its catalog Detection guidance). **No orphans.**
- Primary distribution: M0: 0 (profile/requirements milestone — 3 SUPPORTING appearances) · M1: 15 · M2: 18 · M3: 15 · M4: 29 · M5: 16 · M6: 14 · M7: 7 · M8: 15 · M9: 5.
- Gate classes: **34 HARD · 94 SOFT · 6 ADVISORY** (ADVISORY = the six Low-severity rules; all 11 Critical rules are HARD).
- Applicability: 134 PRIMARY assignments; 52 rules carry SUPPORTING appearances (Round 1 calibration 2026-08-12: CAP-SEC-013 gained a conditional M3 appearance, CAP-ERR-005's M3 appearance removed — net unchanged); 34 rules carry FINAL-GATE re-verifications (several at two milestones); 55 rules carry capability conditions; 25 are runtime-scoped (17 Node.js, 8 Java).
- All 11 Critical rules: HARD, mapped, evidence-defined, M9-visible (§3).
- ORG rules marked `ORG-PENDING`; GEN rules classified by risk (§1.6).
- M0 is deliberately the profile-and-requirements gate (no Layer 2 primaries — its output, the capability profile, is what makes every CONDITIONAL mapping decidable). It is not a placeholder: without an M0 profile, conditional rules are NOT ASSESSABLE.
