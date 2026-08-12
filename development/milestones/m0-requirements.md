# M0 — Requirements & Project Profile

Operational checklist for the [M0 milestone](../lifecycle.md#m0--requirements). Rule mappings: [rule-milestone-matrix](../rule-milestone-matrix.md). Rules are referenced by ID only — the [catalog](../../standards/rules/README.md) is authoritative.

## Purpose
Establish testable requirements **and the capability profile** that makes every CONDITIONAL rule in the matrix decidable. M0 has no PRIMARY Layer 2 rules by design — its deliverable is the input the whole governance system filters on.

## Entry criteria
Project initiated; stakeholders identified; access to source requirements.

## Applicable standards
- **Supporting:** CAP-PRIV-001 (start the personal-data inventory — it becomes M2's annotation baseline), CAP-ARCH-007 *(ORG-PENDING)* (initialize the decision log), CAP-SEC-006 (note the identity-service direction for new projects — decided at M1).
- **Layer 3:** AI-DEV-003 (restate requirements, surface ambiguity) governs how requirements are captured.

## Capability profile (mandatory deliverable)
Determine and record every flag from [matrix §1.4](../rule-milestone-matrix.md#14-capability-profile-drives-conditional-rules): runtime (NODE/JAVA), MT, EVENTING/BROKER, REMOTE/MASHUP, ODATA, DRAFTS, MEDIA, LOCALIZED, TEMPORAL, PERSONAL-DATA, PDM, EXTENSIBILITY/FEATURE-TOGGLES, MCP, IAS/XSUAA, NEW-PROJECT, CF/KYMA, MULTI-LANDSCAPE, UI, plus the workload flags (MASS-DATA, CONCURRENT-EDIT/PAGING, HANA-LARGE, CLOUD-ONLY) as far as knowable — with "unknown, revisit at M1" recorded where genuinely undecided.

## Required evidence
- Requirements document: capabilities, domain glossary, actors/roles, external systems, NFRs (availability, performance targets — ORG G-04), explicit out-of-scope list.
- The capability profile (flags above) with rationale for each non-obvious flag.
- Personal-data inventory (what personal data, which entities will hold it) — or the documented "no personal data" determination.
- Assumptions/open-questions list with owners (AI-DEV-003).
- Decision log initialized (`docs/adr/` or equivalent).

## Required tests
None. Acceptance criteria drafted per requirement (testability check only).

## Review procedure
1. Verify every requirement is uniquely identifiable and testable (has acceptance criteria).
2. Verify the capability profile is complete — every §1.4 flag decided or explicitly deferred with a decision point.
3. Verify NFRs exist for availability/performance/compliance (values are ORG policy — presence is the check).
4. Verify the personal-data determination exists (drives PERSONAL-DATA flag and M2/M6/M9 privacy gates).
5. Verify tenancy and runtime are decided (they scope every runtime-/MT-conditional rule downstream).

## Remediation expectations
Missing profile flags or untestable requirements are completed before M1 — there is no exception path for a missing profile (downstream reviews would be NOT READY).

## Exit criteria (gate result per [matrix §1.3](../rule-milestone-matrix.md#13-milestone-gate-results))
Requirements approved; acceptance criteria exist; capability profile recorded; personal-data determination made; open questions have owners. Result: PASS / NOT READY (no HARD gates exist at M0).
