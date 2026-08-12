# Exception Governance

The exception mechanism lets a project move a HARD-GATE-blocked milestone forward without weakening the underlying rule or silently converting FAIL → PASS. Every gate that PASSes WITH EXCEPTIONS must trace to an approved, in-scope, unexpired record.

Rule reference: [`AI-DOC-002`](../standards/ai/ai-documentation-rules.md) — the exception-record template Claude enforces at review time. Report reference: [templates/review-report-template.md §9](../templates/review-report-template.md). Gate mapping: [rule-milestone-matrix §1.3](../development/rule-milestone-matrix.md#13-milestone-gate-results).

## When exceptions apply

- **HARD-GATE failures** — an approved exception converts a HARD FAIL from `Milestone FAIL` toward `PASS WITH EXCEPTIONS`. Without an exception, one uncovered HARD-GATE violation blocks the milestone.
- **SOFT-GATE failures** — do NOT use the exception mechanism. SOFT findings require an explicit *recorded justification* in the milestone checklist (a lighter artifact); they do not need approver escalation.
- **ADVISORY** — never blocking; never requires an exception.
- **ORG-PENDING rules** — findings are reported as governance-blocking under this standard's own authority, not SAP's; exception governance still applies to their HARD-classified cases.

## Where exception records live

Under the adopting project, not this standards repository:

```
<adopting-project>/
  docs/capm/exceptions/
    <YYYY-MM-DD>-<rule-id>-<short-slug>.md
```

Each file is a single record. The reviewer loads this directory at [review command step 8b](../.claude/commands/capm-review-milestone.md); missing directory is a valid answer (no exceptions on file). Claude Code never creates an exception record on its own — human approval is required.

## Exception record — required fields

```markdown
---
rule: CAP-<CAT>-<NNN>           # affected rule ID (exactly one per record)
status: approved                 # approved | expired | withdrawn
scope:                           # what the exception covers
  services: [ServiceName, …]     # OR entities / operations / paths — be specific
  boundary: |                    # one-paragraph description of the boundary
    <e.g., only the internal admin service; not the customer-facing API>
approver: <name/role>            # the human who approved this
requested_by: <name/role>        # the human who filed it
approved_date: YYYY-MM-DD
expiry: YYYY-MM-DD               # required (see §Expiry)
milestones_covered: [Mx, My]     # the milestone gates this exception applies to
compensating_control: |          # required — how the risk is contained meanwhile
  <the alternative control that reduces residual risk>
re-review_trigger: |             # required — when this record must be re-evaluated
  <specific condition: e.g., "before M6 gate"; "on any change to <file>"; "expiry">
---

# Justification

<the business/technical reason the rule cannot be satisfied here — MUST be concrete>

# Residual risk

<what could still go wrong; the honest assessment>

# Alternatives considered

<what full-compliance options were considered and why they were rejected>
```

**No field is optional.** A record missing any of `rule`, `approver`, `expiry`, `scope`, `compensating_control`, or `re-review_trigger` is treated as no record — the underlying finding stands.

## Who can approve

Approval is an organizational responsibility, not a technical one. This document names the roles the standard expects; the specific individuals belong in the organization's own policy.

| Rule severity | Approver role required |
|---|---|
| **Critical** (11 rules — CAP-EVT-002/-003, CAP-MT-003/-005, CAP-SEC-001/-002/-003/-005/-009/-013/-015) | Security lead + engineering leadership (at least two approvers; recorded as `approver: <person1>, <person2>`) |
| **High + HARD** | Engineering lead or above |
| **High + SOFT** | Team lead (recorded justification, not exception — see above) |
| **Medium / Low** | Handled as recorded justifications, not exceptions |

Approver names/roles are recorded verbatim in the exception record; a review verifies the recorded approver matches an actual person authorized by the organization's ratified policy.

## Critical rules — additional requirements

For any of the 11 Critical rules, the exception record MUST additionally contain:

- Explicit statement: *"This exception acknowledges a Critical rule is not satisfied and documents the accepted risk."*
- A named, dated compensating control that is itself reviewable (a network policy, a WAF rule, a monitoring alert — not "we are careful").
- Two approvers (see table above).
- Expiry ≤ 6 months; renewals require new evidence and a fresh approval, not just a date bump.

A Critical exception never silently persists across major CAP releases; the [re-verification cadence](re-verification-cadence.md) requires re-evaluation on each release.

## Expiry

- **Every** exception carries an expiry date. Records without one are invalid.
- Reviews performed after `expiry` treat the record as if it did not exist. The finding stands; no partial credit.
- Renewal = new record with a new date and fresh approval, not editing the old one. History is preserved.
- Expired records are moved (not deleted) to `docs/capm/exceptions/expired/`.

## Scope

Scope is what the exception covers, and only that:

- Named services / entities / operations / file paths — not vague ("the admin area").
- If a code change would move the risk to a different subject not named in scope, the exception does not cover the new subject.
- Scope changes require a new record (not an edit).

## Re-review requirements

- An exception record is loaded at every review that evaluates its `milestones_covered`.
- On re-review after remediation, the exception is still evaluated: if the underlying finding is now genuinely resolved, the exception is superseded (record `status: withdrawn`, keep the file).
- HARD-gate exceptions are re-evaluated at every subsequent milestone the rule appears at (PRIMARY, SUPPORTING with owning-milestone visibility, FINAL-GATE). A rule that was excepted at M3 does not automatically stay excepted at M6/M8/M9.
- Cross-cutting security observations (see [review command step 9a](../.claude/commands/capm-review-milestone.md)) are NOT exceptions; carrying an observation across reviews is a workflow feature, not a grant of exception status.

## What exceptions can NOT do

- **Convert a finding to PASS.** They convert the milestone gate from `FAIL` toward `PASS WITH EXCEPTIONS`. The per-rule verdict stays FAIL (report §8), the finding stays in the report (§3 + §9).
- **Downgrade a rule.** Severity/authority/detection are properties of the rule, not the record.
- **Cover future risk unspecified today.** Scope is what's named.
- **Apply to another project.** Records are per-project artifacts.
- **Waive Critical rules indefinitely.** The 6-month cap forces revisiting.

## Review-time verification (what Claude Code does at command step 8b)

1. Load `docs/capm/exceptions/` (missing directory = no exceptions).
2. For each finding that would trigger HARD-GATE FAIL, look for a record whose `rule` matches.
3. Verify: `status: approved`, `expiry` in the future, `scope` covers the specific evidence of the finding, `milestones_covered` includes the current milestone, all required fields present.
4. If verified: gate moves toward `PASS WITH EXCEPTIONS`; the report §9 lists the record with a link to it.
5. If not verified (expired, out-of-scope, wrong milestone, missing field): the record is rejected with the specific reason in §9; the finding still blocks.

**Claude Code never approves an exception, never creates a record, never edits a record.** Verification is read-only; approval is human.

## Auditability

- Every gate result of `PASS WITH EXCEPTIONS` traces to one or more records that a human reviewer can independently verify against the organization's approval log.
- The [review report](../templates/review-report-template.md) §9 lists every accepted and every rejected record with the verification reason.
- The organization's overall exception posture (how many exceptions, which rules, which projects, expiring when) is a governance metric the [re-verification cadence](re-verification-cadence.md) tracks annually.

## What this document is not

Not the rule catalog (see [`standards/rules/`](../standards/rules/README.md)). Not the gate model (see [rule-milestone-matrix.md §1.3](../development/rule-milestone-matrix.md#13-milestone-gate-results)). Not the AI-DOC rule definition itself (see [`AI-DOC-002`](../standards/ai/ai-documentation-rules.md)). This document is the operational contract between the exception mechanism and the review workflow.
