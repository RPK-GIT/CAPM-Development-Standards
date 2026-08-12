# Adoption Boundaries — What v1.0 Does NOT Guarantee

This document is a promise about what CAPM Development Standards v1.0 **does not** claim, cover, or provide. It is intended to be read alongside the [adoption guide](adoption-guide.md) and the [executive overview](executive-overview.md).

If a stakeholder infers any of the guarantees listed below from a v1.0 review report, the review was mis-communicated. The reviewer's responsibility is to write reports that do not license these inferences (see [AI-DOC-004](../standards/ai/ai-documentation-rules.md) and the scope-statement discipline in [templates/review-report-template.md](../templates/review-report-template.md)).

## Guarantees NOT made

### The standard does not guarantee defect-free software.
A PASS milestone gate means every applicable rule in the milestone's evaluated set was satisfied against the evidence available at review time. Rules cover the failure modes SAP documents plus a defined set of organizational risks — not the universe of possible defects. Functional correctness, business-logic completeness, performance under load, and third-party integration behaviors are outside the rule catalog's scope except where explicit rules address them.

### The standard does not replace SAP documentation.
`cap.cloud.sap/docs/` remains the primary authority for CAP mechanics. Rules cite SAP sources; they do not substitute for them. Where a rule and its cited SAP page disagree, the SAP page wins and the rule is a [governance case](rule-governance.md) for re-verification.

### The standard does not replace security testing.
- No rule replaces a penetration test.
- No rule replaces a threat model.
- No rule replaces an audit against organizational security policy or against regulatory frameworks.
- Security testing (SAST, DAST, dependency vulnerability scanning) is called out as CI/CD scope ([CAP-CICD-003](../standards/rules/cicd.md), currently ORG-PENDING); wiring, executing, and interpreting those tools remains the organization's responsibility.

### The standard does not replace runtime testing.
- Rules inspect the code, the model, the configuration — they do not exercise the running system.
- Round 3 of the pilot ([`pilots/pilot-1-scs-m3-round-3/review-report.md`](../pilots/pilot-1-scs-m3-round-3/review-report.md)) verified authorization annotations at compile-time; it did not verify runtime allow/deny behavior, and it said so explicitly. Runtime allow/deny testing belongs to [CAP-TEST-*](../standards/rules/testing.md) at M6/M7.
- Load testing, chaos testing, and observability validation are the project's operational responsibility ([CAP-OPS-*](../standards/rules/production-readiness.md)), not surrogate outputs of a review.

### The standard does not replace architecture review.
- Rules constrain individual decisions ([CAP-ARCH-001](../standards/rules/architecture.md) — CAP-native first; [CAP-ARCH-003](../standards/rules/architecture.md) — single-use-case services; …) — they do not audit the whole architecture.
- Cross-cutting design concerns (system topology, data-flow assumptions, capacity planning, resilience patterns) sit outside a rule catalog and require human architectural review.

### The standard does not replace organizational policies.
- HR policy, legal review of terms/data-processing agreements, procurement approval of external services, and vendor-security reviews are organizational responsibilities.
- Where the CAPM standard names something as ORG (currently [CAP-ARCH-007, CAP-CICD-003, CAP-OPS-003](org-ratification.md)), the standard defers to the organization's ratified policy — it does not create that policy.

### CAPire sources can change.
- Official SAP documentation is updated on release cadence.
- The [CAPire verification protocol](../reviews/capire-verification.md) catches material source changes at review time — but it is a review-time check, not a real-time subscription.
- A rule whose source has materially changed becomes `NOT ASSESSABLE`, not silently PASS. Reviewers must re-fetch the sources listed in the [source map](../references/sap-cap-sources.md) per the [re-verification cadence](re-verification-cadence.md).

### Rules require periodic re-verification.
- The `Last verified: YYYY-MM-DD` field on every rule declares when its SAP evidence was last confirmed live.
- A rule with a stale verified-date is not automatically wrong — but it is not automatically right either. The [quarterly cadence](re-verification-cadence.md) exists for this reason.
- Between cadence sweeps, individual reviews re-verify only the sources they cite; sources they do not cite may drift silently until the next sweep.

### AI-generated implementation still requires human accountability.
- Claude Code operates under Layer-3 rules ([standards/ai/](../standards/ai/README.md)); these constrain its behavior but do not remove the need for a human owner of the code.
- Every change Claude authors — annotations, aspects, handlers, tests — enters the same review path as human-authored changes, subject to the same rules.
- Approvals — of exceptions, of ORG ratifications, of remediation-plan sign-offs — are human roles. Claude never approves.
- A completion report from `/capm-develop` is a self-assessment; it does not substitute for the independent review at `/capm-review-milestone`.

### A PASS is not a certification.
- v1.0 is not issued by SAP and is not endorsed by SAP. Reports must never claim "SAP certified", "SAP approved", or equivalent.
- The correct scope claim in every report is: *"passed the applicable Mx standards evaluated in this review"* — not "complies with CAP best practices" and not "production-proven". This is enforced by [AI-DOC-004](../standards/ai/ai-documentation-rules.md).

### One project's pilot does not generalize.
- v1.0's validation covered one live project across three review rounds ([Pilot 1](../pilots/pilot-1-scs-m3/README.md) → [Round 1 calibration](../pilots/round-1-calibration.md) → [Round 2](../pilots/pilot-1-scs-m3-round-2/README.md) → [Round 3 remediation](../pilots/pilot-1-scs-m3-round-3/README.md)) plus 36 pre-pilot fixture scenarios and 8 additional post-calibration scenarios ([validation results](validation-results-2026-08.md)).
- One Node.js project, one milestone deeply reviewed (M3), one small controlled remediation. The framework demonstrated its closed loop on real code — that does not equate to broad multi-project or multi-milestone experience.
- Broader adoption confidence requires broader pilot uptake — which is the point of ratification and the recommended [rollout path](adoption-guide.md#next-steps).

### Exceptions transfer risk. They do not remove it.
- An approved exception moves a milestone gate from `FAIL` toward `PASS WITH EXCEPTIONS`. The per-rule verdict stays `FAIL`. The residual risk is documented, not eliminated.
- Every exception carries an expiry. On expiry, the finding blocks again.
- Compensating controls named in exception records are themselves subject to review — they must be real, dated, and evaluable.

### A NOT-APPLICABLE verdict is a statement of scope.
- A rule marked `NOT APPLICABLE` for this project (e.g., `CAP-MT-*` on a single-tenant project) says: this rule does not apply to this project's shape as declared in the profile.
- If the profile is wrong (`multitenant: false` when the project is in fact multitenant), the `NOT APPLICABLE` verdict is wrong. The remedy is to correct the profile, then re-review — not to argue with the rule.

### A NOT-ASSESSABLE verdict is a statement of evidence.
- `NOT ASSESSABLE` says: the rule's detection procedure needs evidence this repository does not carry.
- It is not a soft PASS. It blocks reliable milestone evaluation until the evidence is produced (owner input on a workload flag, a test-suite run, a config export from the runtime).
- Silently converting `NOT ASSESSABLE` to `NOT APPLICABLE` is a workflow violation (AI-REVIEW-010).

## What v1.0 does provide

The other side of the ledger is not this document's subject — see the [executive overview](executive-overview.md) for a positive statement, or the [README](../README.md) for the concise version. In short: a versioned, evidence-backed catalog of 134 rules mapped to a defined lifecycle, with two Claude Code commands, an evidence-verification protocol, and demonstrated closed-loop behavior on real code.

## Reading a report against these boundaries

When you read a review report:

- Look at the header's *Scope* and *Review mode* fields.
- Look at §2 counts: PASS is against the *evaluated set*, not the whole catalog.
- Look at §7 (Applicability decisions): what was `NOT APPLICABLE` and why.
- Look at §6 (NOT ASSESSABLE): what could not be evaluated.
- Look at §10 (CAPire): what SAP evidence was verified and when.
- Look at §9 (Exceptions): what risk was accepted rather than resolved.

A report that omits any of these sections is incomplete. A summary that omits them is misleading.

## Where to go next

- Positive positioning → [executive-overview.md](executive-overview.md).
- Practical use → [developer-guide.md](developer-guide.md), [reviewer-guide.md](reviewer-guide.md).
- What must still be decided → [org-ratification.md](org-ratification.md).
- How the standard stays current → [re-verification-cadence.md](re-verification-cadence.md).
