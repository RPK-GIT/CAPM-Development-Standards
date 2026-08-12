# ORG Ratification — Pending Decisions for v1.0

Three rules in the Layer 2 catalog carry `ORG-PENDING` status: **CAP-ARCH-007**, **CAP-CICD-003**, **CAP-OPS-003**. They function as organization policy in v1.0 reviews (findings are reported as *ORG-PENDING policy finding — blocking authority derives from this standard's own governance, not SAP; subject to ratification*), but they require formal ratification by the adopting organization before they carry that authority in production posture reports.

**These records are NOT marked ratified.** The `Ratified: date` and `Effective: date` fields are placeholders for organizational sign-off. Until filled, reports MUST cite these rules with the ORG-PENDING language quoted above ([AI-DOC-004](../standards/ai/ai-documentation-rules.md); [matrix §1.6](../development/rule-milestone-matrix.md#16-org-and-gen-rules)).

**Under no circumstance are these rules presented as SAP requirements.** SAP has not issued guidance on these subjects. They are legitimate organizational-policy choices, and this document records them as such.

---

## CAP-ARCH-007 — Documented architecture decision records (ADRs)

**Decision required:** Should every significant architectural decision (service split, external-service integration, database choice, custom framework rejection, tenant model change) be captured as an ADR under `docs/adr/` in the adopting project's repository?

**Rationale:**
- SAP documents CAP architecture patterns extensively but does not require ADRs.
- Without an organizational policy on ADRs, reviewers have no consistent basis to flag missing decision records.
- The rule as written (see [architecture.md CAP-ARCH-007](../standards/rules/architecture.md)) requires ADRs — but this is an organizational judgment call, not a SAP requirement.

**Proposed policy:**
> ADRs are required for: (a) service boundary decisions affecting more than one team, (b) integration of a new external system or S/4 endpoint, (c) departure from CAP-native mechanisms (custom pagination, custom auth framework, hand-built draft handling), (d) tenant-model choices, (e) database choices. Format: an ADR template defined by the organization at ratification time (a one-page template is sufficient) — one file per decision under the adopting project's `docs/adr/`, immutable once approved (supersession by new ADR, not edit). Missing ADR where required → CAP-ARCH-007 finding.

**Impact of ratification:**
- Adopting projects add `docs/adr/` and follow the ADR template.
- Reviewers gain a defined artifact to check under CAP-ARCH-007.
- Existing projects: a grace period should be defined (recommend: 6 months from ratification date; existing decisions may be back-filled).

**Approval owner:** *[placeholder — engineering leadership]*
**Effective date:** *[placeholder — YYYY-MM-DD]*
**Ratified:** *[placeholder — pending organizational decision]*

**Gate class if ratified:** SOFT (SAP-silent; failure mode is process/traceability, not correctness or security). Retain PRIMARY at M1, FINAL-GATE at M9 per current mapping.

---

## CAP-CICD-003 — Pipeline security gates

**Decision required:** Should CI/CD pipelines run automated security checks (dependency vulnerability scan, SAST/DAST as applicable, secret detection) and block merge on findings above a defined threshold?

**Rationale:**
- SAP recommends secure development but does not prescribe pipeline configuration.
- The rule ([cicd.md CAP-CICD-003](../standards/rules/cicd.md)) requires that security gates exist, without prescribing specific tools or thresholds.
- Tool selection and threshold policy are organizational decisions.

**Proposed policy:**
> Every CAP application's CI pipeline includes: (i) SBOM/dependency-vulnerability scan against a maintained CVE feed, blocking on unresolved High/Critical vulnerabilities in production dependencies; (ii) secret detection on commits (pre-receive or pipeline stage); (iii) where the project has HTTP surface, a per-release DAST or equivalent for the top OWASP categories. Specific tool names are recorded in the organization's CI standards, not in this repository. Missing gates or bypassed gates → CAP-CICD-003 finding.

**Impact of ratification:**
- Adopting projects wire the named gates into their pipeline.
- Reviewers check pipeline configuration for the named gates (evidence: Jenkinsfile, GitHub Actions workflow, Azure Pipelines YAML — the reviewer inspects the file).
- Tool inventory sits in the organization's CI standards, not the CAPM standard — this keeps the CAPM rule tool-agnostic.

**Approval owner:** *[placeholder — security lead + platform/CI owner]*
**Effective date:** *[placeholder — YYYY-MM-DD]*
**Ratified:** *[placeholder — pending organizational decision]*

**Gate class if ratified:** SOFT (organizational process; individual project may proceed with recorded justification if the pipeline is genuinely not applicable, e.g., pre-M8 exploration). Retain PRIMARY at M8, FINAL-GATE at M9 per current mapping.

---

## CAP-OPS-003 — Operational-readiness assessment

**Decision required:** Should every CAP application produce an operational-readiness assessment before go-live, covering monitoring, alerting, on-call, incident response, and post-mortem discipline?

**Rationale:**
- SAP documents operational aspects of CAP (`cds.log`, application logs, health endpoints) but does not require an operational-readiness artifact.
- The rule ([production-readiness.md CAP-OPS-003](../standards/rules/production-readiness.md)) requires the assessment; the assessment itself is an organizational deliverable.

**Proposed policy:**
> Before a CAP application transitions to `project.milestone: LIVE` (M9 gate PASS), the project produces an operational-readiness assessment covering: (a) monitoring — what metrics are watched, where dashboards live; (b) alerting — SLO thresholds and paging destinations; (c) on-call — coverage schedule and escalation path; (d) incident response — runbook location; (e) post-mortem — how post-incident learning is captured. Template: an operational-readiness template defined by the organization at ratification time (a one-page checklist per (a)–(e) is sufficient). Missing assessment or missing coverage of any of (a)–(e) → CAP-OPS-003 finding at M9.

**Impact of ratification:**
- Adopting projects add the assessment as an M9 exit artifact.
- Reviewers evaluate the artifact at M9 FINAL-GATE.
- Ongoing operations — the assessment is a living document; a periodic refresh cadence (recommend: annually or after material architectural change) sits in the organization's operational standards.

**Approval owner:** *[placeholder — operations lead + engineering leadership]*
**Effective date:** *[placeholder — YYYY-MM-DD]*
**Ratified:** *[placeholder — pending organizational decision]*

**Gate class if ratified:** SOFT (the standard already carries HARD-gate rules for the technical prerequisites — CAP-LOG-005, CAP-SEC-*, etc. — this rule enforces the *artifact* documenting them). Retain PRIMARY at M9.

---

## Ratification process

The adopting organization ratifies these rules by:

1. Reviewing the proposed policy above and adjusting scope, thresholds, and template names as appropriate for the organization.
2. Naming the approval owner for each rule.
3. Setting the `Effective date` — projects starting after that date are subject to the rule at its full gate class; projects in flight may negotiate a grace period.
4. Filling in `Ratified: <date>` here and updating the rule's `Status` field in the catalog from `Active (ORG-PENDING)` to `Active`.
5. Announcing the ratification in [CHANGELOG.md](../CHANGELOG.md).

Ratification is an organizational decision — Claude Code does not ratify, and this repository does not carry any actor authorized to ratify on the organization's behalf.

## Until ratified

- Reports state: *"ORG-PENDING policy finding — blocking authority derives from this standard's own governance, not SAP; subject to ratification."*
- The rule still fires in review (findings are visible, not silently suppressed).
- The gate blocks per the rule's gate class — organizational governance, not SAP authority, is the blocking basis.
- No project should skip these three rules because they are ORG-PENDING; the correct posture is to satisfy them (they represent legitimate defensive practice), or to record an [exception](exception-governance.md) with organizational approval — same discipline as any HARD-gate rule.

## Where to go next

- Authority model → [authority-levels.md](authority-levels.md).
- Matrix §1.6 (ORG/GEN handling) → [../development/rule-milestone-matrix.md § 1.6](../development/rule-milestone-matrix.md#16-org-and-gen-rules).
- The three rules → [architecture.md CAP-ARCH-007](../standards/rules/architecture.md) · [cicd.md CAP-CICD-003](../standards/rules/cicd.md) · [production-readiness.md CAP-OPS-003](../standards/rules/production-readiness.md).
