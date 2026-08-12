# CAPM Development Standards — Executive Overview (v1.0)

**Audience:** delivery leads, engineering managers, architects, and sponsors of teams building applications on the SAP Cloud Application Programming Model (**CAP** — SAP's official framework for building enterprise services on SAP Business Technology Platform).

**One-line positioning.** The CAPM Development Standards are a single, versioned, evidence-backed engineering standard for how our organization designs, builds, tests, secures, deploys, operates, and reviews CAP applications — usable by human developers and by [Claude Code](../CLAUDE.md) (Anthropic's coding agent) as both an implementation assistant and an independent reviewer.

---

## 1. Why this standard exists

CAP is powerful and evolving. Teams starting a project today face three recurring risks:

1. **Divergence from SAP guidance.** Official CAP documentation ("**capire**", at `cap.cloud.sap/docs`) is broad and moves quarterly. Individual teams read it selectively; opinions drift and are then presented as vendor requirements.
2. **Late-stage security and multitenancy defects.** Authorization, tenant isolation, injection-safe queries, secrets handling, and production auth configuration are routinely discovered at the end of a project — the most expensive place to find them.
3. **Inconsistent AI usage.** Coding agents accelerate delivery, but without a shared operating contract they invent abstractions that duplicate CAP capabilities, silently weaken security, or "self-attest" compliance without evidence.

The standard addresses these directly: one authoritative rule catalog per project family, one lifecycle, one review protocol, one set of rules governing how AI must behave. Positioning and scope: [README.md](../README.md).

---

## 2. What the standard is

Three layers, one identifier scheme, one lifecycle. Full design: [docs/standard-architecture.md](standard-architecture.md).

| Layer | What it contains | Where |
|---|---|---|
| **1 — Engineering Principles** | ~10 fundamental principles (domain-first modelling, CAP-native-first, secure-by-design, testable behaviour) | [../standards/principles/](../standards/principles/engineering-principles.md) |
| **2 — Engineering Rules** | 134 formal rules across 20 categories (architecture, CDS modelling, services/APIs, security, multitenancy, testing, deployment, operations, …) | [../standards/rules/README.md](../standards/rules/README.md) |
| **3 — AI Development & Review Rules** | Rules governing Claude Code's own behaviour when it develops or reviews CAP code | [../standards/ai/README.md](../standards/ai/README.md) |

Distribution of the 134 Layer-2 rules: **34 HARD-gate** (must be met to progress), **94 SOFT-gate** (must be met or explicitly justified), **6 ADVISORY**. All **11 Critical rules are HARD** and re-verified at the final production-readiness gate. Full mapping: [../development/rule-milestone-matrix.md §5](../development/rule-milestone-matrix.md).

### Authority is always labelled

Every statement carries one of five authority levels — **SAP-REQ** (SAP documents it as a requirement), **SAP-REC** (SAP recommends it), **GEN** (general engineering practice), **ORG** (our organization's policy, filling a gap where SAP is silent), **AI-REC** (Claude recommendation, not yet ratified). This prevents the single most damaging failure mode of an engineering standard — presenting opinion as vendor requirement. Definitions: [docs/authority-levels.md](authority-levels.md).

---

## 3. How CAPire evidence is used

Every SAP-authority statement in the standard cites a specific official SAP page and carries a `Last verified` date. Evidence lives separately from norms (in [`../references/`](../references/sap-cap-sources.md)) so that we can re-verify against a new CAP release without rewriting rules.

At review time, before the report is finalized, the reviewer performs a **live CAPire source check** against only the pages relevant to the rules actually evaluated in that review — three verification levels, with **Critical / security / privacy** and **version-sensitive** rules receiving explicit page-level re-fetch. If a source has moved, materially changed, or been removed and the rule depends on it, the verdict becomes `NOT ASSESSABLE` — never a silent pass. Full protocol: [../reviews/capire-verification.md](../reviews/capire-verification.md).

The output of every review carries the mandatory traceability chain:

`Rule → Project evidence → CAPire source → Current source status → Verdict`

---

## 4. How Claude Code is used

The standard defines two operational modes for Claude Code, each triggered by an explicit slash-command:

- **Develop** (`/capm-develop`) — Claude builds against a project-specific profile, uses only the subset of rules applicable to the current milestone (never all 134), prefers CAP-native capabilities over custom code, and produces a completion report with per-rule evidence.
- **Review** (`/capm-review-milestone <Mx>`) — Claude performs an **independent, read-only** review of the actual repository, filtered by the same profile and milestone, verifies CAPire sources live, and produces a structured report with verdicts of `PASS` / `FAIL` / `NOT APPLICABLE` / `NOT ASSESSABLE` per rule — never impressionistic.

Layer 3 rules constrain Claude's behaviour explicitly: no silent standard-weakening, no self-attestation without evidence, no bypassing CAP abstractions, no disabling tests to make them pass. Operating contract: [../CLAUDE.md](../CLAUDE.md).

---

## 5. The M0–M9 lifecycle in one paragraph

Every project passes through ten milestones — **M0** requirements and capability profile, **M1** architecture, **M2** domain/CDS model, **M3** services and initial authorization surface, **M4** business logic / error handling / core security (injection-safe queries), **M5** integration and eventing, **M6** production security posture, **M7** testing, **M8** deployment configuration, **M9** production readiness — each with a defined subset of rules in scope for its quality gate, and a `PASS` / `PASS WITH EXCEPTIONS` / `FAIL` / `NOT READY` outcome. Security rules deliberately recur across M3, M4, M6, M8, and M9, so a clean M6 does not immunize later changes. Full lifecycle: [../development/lifecycle.md](../development/lifecycle.md) · rule-to-milestone mapping: [../development/rule-milestone-matrix.md](../development/rule-milestone-matrix.md) · per-milestone checklists: [../development/milestones/m0-requirements.md](../development/milestones/m0-requirements.md).

---

## 6. Benefits

**For developers.** One authoritative place to answer "what does good CAP look like here?" Rules carry good and bad code examples, so they are constructive, not just evaluative. Only the milestone's rule subset applies at any moment — the largest single-milestone rule load is 29 rules at M4, and typically drops to ~21 after profile filtering. AI assistance operates under the same contract as human developers, removing the "the agent did it differently" tax.

**For reviewers, architects, and sponsors.** Reviews are deterministic, per-rule, and evidence-cited (file:line where possible). Findings distinguish defects (rule violations) from recommendations. Every finding names its authority — so an "SAP requires" finding cannot be confused with an "our organization decided" finding or an "AI suggests" finding. Milestone gates make risk visible at the point it is cheapest to remediate. Security and multitenancy remain visible across the whole lifecycle by design.

---

## 7. Governance model

- **Rule lifecycle.** Rules are modular Markdown files with **stable IDs** (`CAP-SEC-001`, `AI-REVIEW-004`, …), never renumbered or reused. Rules are added, amended, or `Deprecated` (with a successor pointer) — never silently deleted. Every change to a rule is a diff against a single file with an updated `Last verified` date. Templates: [../templates/rule-template.md](../templates/rule-template.md).
- **Exceptions.** Any HARD-gate violation may only progress the milestone via an `AI-DOC-002` **approved exception** that records the rule, deviation, impact, compensating controls, approver, and expiry/review condition. Exceptions are explicit and auditable; there is no such thing as an unrecorded pass.
- **ORG ratification.** Three rules are currently marked `ORG-PENDING` (CAP-ARCH-007, CAP-CICD-003, CAP-OPS-003). They function as ORG policy but must be presented in reports as *"ORG-PENDING policy finding — subject to ratification"*, never as SAP requirements. Ratification is an organization decision, not a Claude decision.
- **Version resilience.** CAP evolves quarterly. The standard binds rules to CAP-version applicability and re-verifies sources on a quarterly cadence. Version policy: [docs/version-management.md](version-management.md).

---

## 8. What v1.0 has been validated against

v1.0 has been **validated through controlled real-project pilot and remediation experiments (Pilot 1 · Rounds 1–3 on a live BTP project, `scs`)**:

- **Round 1** — first live M3 review against a real Node.js CAP project. Result: AMBER. Two rule detection amendments, one rule scope clarification, review-mode formalization, evidence-state formalization, finding consolidation, and cross-cutting security observation mechanism introduced. See [../pilots/round-1-calibration.md](../pilots/round-1-calibration.md).
- **Round 2** — re-review after standard calibration. Result: AMBER (closed loop half-validated). See [../pilots/pilot-1-scs-m3-round-2/README.md](../pilots/pilot-1-scs-m3-round-2/README.md).
- **Round 3** — controlled remediation experiment: two genuine `FAIL → PASS` transitions (Critical HARD `CAP-SEC-001`, High SOFT `CAP-SEC-010`); nine unremediated FAILs correctly preserved; HARD gate cleared. Result: **GREEN**. See [../pilots/pilot-1-scs-m3-round-3/README.md](../pilots/pilot-1-scs-m3-round-3/README.md).

Additionally, the rule catalog was operationally validated **36/36** across Node.js and Java fixtures ([../docs/validation-results-2026-08.md](validation-results-2026-08.md)) before the pilots began.

---

## 9. Limitations

Adopters should read v1.0 with these boundaries in mind:

1. **Pilot breadth is narrow.** Validation to date is one real project (Node.js on SAP BTP), three review rounds, one Java scenario suite. Java-runtime coverage in live projects and multitenant SaaS scenarios remain under-piloted.
2. **`ORG-PENDING` rules are not yet ratified.** Three rules (CAP-ARCH-007, CAP-CICD-003, CAP-OPS-003) function as organization policy in v1.0 but require formal ratification. Reports must present them accordingly.
3. **No certification claim.** This standard is neither issued by SAP nor endorsed by SAP. Compliance with this standard is not equivalent to SAP compliance. SAP-authority findings cite SAP documentation; ORG/GEN/AI-REC findings do not.
4. **A review is not a guarantee.** A PASS verdict means every evaluated rule was satisfied against the evidence available at review time. It does not eliminate the possibility of defects, regressions, or vulnerabilities outside the rule catalog's scope. Exceptions transfer risk; they do not remove it.
5. **CAP evolves.** Rules are version-scoped and quarterly-reverified. Between verification sweeps, a rule's cited SAP page may shift; the CAPire live-check protocol ([../reviews/capire-verification.md](../reviews/capire-verification.md)) is what catches this at review time.
6. **AI usage requires discipline.** Layer 3 rules constrain Claude Code, but they do not eliminate the need for human review of consequential changes — architectural decisions, security posture, tenant-isolation code, and production configuration should always receive human sign-off.
7. **The standard is documentation.** It does not run tests, deploy code, or block pipelines by itself. It defines what "done" looks like at each milestone; enforcement remains an organizational responsibility.

---

## 10. Where to go next

- Adopt in a project → [../CLAUDE.md § Adoption](../CLAUDE.md) · profile template: [../templates/capm-profile-template.yaml](../templates/capm-profile-template.yaml)
- Understand the rule catalog → [../standards/rules/README.md](../standards/rules/README.md)
- Understand how reviews work → [../reviews/review-model.md](../reviews/review-model.md)
- See a worked review → [../examples/worked-example-m3/README.md](../examples/worked-example-m3/README.md) (illustrative / non-production)
- See a real pilot → [../pilots/pilot-1-scs-m3-round-3/review-report.md](../pilots/pilot-1-scs-m3-round-3/review-report.md)
