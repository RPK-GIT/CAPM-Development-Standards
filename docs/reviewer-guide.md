# Reviewer Guide

Practical usage of CAPM Development Standards v1.0 for reviewers evaluating CAP applications. Companion to [/capm-review-milestone](../.claude/commands/capm-review-milestone.md), the [review model](../reviews/review-model.md), and the [AI-REVIEW rules](../standards/ai/ai-review-rules.md).

## Reviewer responsibilities

Reviews are **read-only** (AI-REVIEW-012). The reviewer does not modify application code, does not "fix on behalf of", and does not accept developer assertions as evidence unless a rule explicitly permits a documented decision. Reviews produce structured evidence, not opinions.

- Follow the [/capm-review-milestone procedure](../.claude/commands/capm-review-milestone.md) in order.
- Cite file:line evidence for every finding (AI-REVIEW-001).
- Never impressionistic verdicts — exactly one of `PASS` / `FAIL` / `NOT APPLICABLE` / `NOT ASSESSABLE` per applicable rule (AI-REVIEW-002/-010).
- Distinguish defects (rule violations) from recommendations (AI-REVIEW-003).
- Never claim SAP requires something unless the rule is `SAP-REQ` with a verified source (AI-REVIEW-006).
- Never silently downgrade a HARD-gate violation — exception records go through the review flow with reasons in report §9.

## Step-by-step

### 1. Load and validate the profile

`capm-profile.yaml` at the project root. Validation per [project-profile.md § Validation procedure](../development/project-profile.md#validation-procedure-executed-by-both-commands-before-any-rule-work). Missing required fields → `NOT-ASSESSABLE` naming exactly what's missing. Contradictions between profile and repository → `CONTRADICTORY profile evidence`; the review stops for correction.

Never guess missing values. You may *propose* a value derived from repository inspection (e.g., runtime from `package.json`); it enters the profile only by user acceptance.

Workload flags valued `unknown` — the rules conditioned on them are `NOT ASSESSABLE` naming the flag. Not silently `NOT APPLICABLE`. This is [Round 1 calibration WCP-3](../pilots/round-1-calibration.md) behavior.

### 2. Determine the review mode

- **DEVELOPMENT** — the reviewed milestone is what the project is currently developing (`project.milestone` equals it, application not yet complete).
- **RETROSPECTIVE** — the reviewed milestone lies behind the project's actual position (profile milestone later, or `LIVE`).
- `LIVE` additionally means the application is deployed — findings represent **current operational risk**, not historical/pre-release risk.

The report header states the mode.

### 3. Filter the rule set

Never load all 134 rules. Selection per [command step 5](../.claude/commands/capm-review-milestone.md):

- PRIMARY + FINAL-GATE rows for the milestone.
- SUPPORTING rows:
  - DEVELOPMENT mode: rows whose subject the milestone's changes touch.
  - RETROSPECTIVE mode: evidence-driven — a row is selected iff (a) the profile declares the driving capability, or (b) the rule's *Evidence expected in code* artifacts exist in the repository, or (c) a PRIMARY rule of this review produced a finding listing it under *Related rules*. Never all-by-default; never outside the milestone's checklist supporting list. Record the per-row (a)/(b)/(c) rationale in report §7.
- Then filter by runtime (Java-only rule never appears in a Node.js review), capability flags, CAP version against the [register](version-management.md), and deployment target where relevant.
- Note each rule's gate class (HARD/SOFT/ADVISORY) from the matrix.
- Mark ORG rules `ORG-PENDING` per matrix [§1.6](../development/rule-milestone-matrix.md#16-org-and-gen-rules).

### 4. Collect evidence

Per each rule's *Detection guidance*. Open the actual files (AI-REVIEW-001) — the review is against the actual repository state, not a summary of it.

Classify evidence:

- `DIRECT` — demonstrates compliance (annotation present at file:line).
- `INDIRECT` — supports, not definitive (config likely enforcing it, verified elsewhere).
- `MISSING` — expected artifact absent (verified absence = evidence; still a determinable answer).
- `CONTRADICTORY` — indicates violation (config says X, code says Y).
- `NOT-ASSESSABLE` — repository lacks sufficient evidence and no accepted decision-record substitute; the rule permits no verdict.

Developer assertions are not evidence unless a rule explicitly accepts a documented decision (`CAP-SRV-008`'s recorded last-write-wins is one).

### 5. CAPire verification

Per [capire-verification.md](../reviews/capire-verification.md). Only sources relevant to the evaluated rules; one fetch per unique URL.

- **L1** — URL liveness for orientation.
- **L2** — explicit fetch, verbatim re-confirmation of the load-bearing sentence. Required for Critical / security / privacy / tenant-isolation rules.
- **L3** — L2 + version-register cross-check. Required for version-sensitive rules.

Source-status effects (per protocol):

- `CURRENT` — verdicts stand.
- `CURRENT BUT EVOLVING` — verdicts stand; observation noted for the register.
- `REDIRECTED` — follow the redirect; treat as new URL and re-verify.
- `STALE` — rule is materially dependent on the source and the source has drifted → verdict downgrades to `NOT ASSESSABLE`; governance flag raised.
- `REMOVED` (404, no successor) — same as STALE, unless the rule anchors a live alternative.
- `UNAVAILABLE` (offline) — verdict stands only if the [register](version-management.md) is within cadence; report states "live check unavailable; relying on register verification of <date>".

**Never silently reuse Round-1's verification in Round-2.** The command explicitly requires fresh fetches (with a same-day pool acceptable when documented).

### 6. Evaluate rules and produce per-rule verdicts

For each selected rule → exactly one verdict. Insufficient evidence → `NOT ASSESSABLE`, never converted to PASS (AI-REVIEW-010). Findings that "seem" wrong but the detection doesn't fire → NOT a defect for this rule — record as observation in §5 if serious.

### 7. Critical rules — prominence

For every applicable Critical rule (11 total; matrix [§3](../development/rule-milestone-matrix.md)):

- Verify evidence + source + runtime + version applicability **explicitly**.
- Evaluate as HARD-GATE regardless of milestone class.
- Surface the result in report §3 as its own subsection with the full evidence chain.
- Insufficient evidence → `NOT ASSESSABLE`, stated as such. Never silently skipped.

### 8. Root-finding consolidation

Per [report-template §3 convention](../templates/review-report-template.md) and [command step 13](../.claude/commands/capm-review-milestone.md):

When multiple rules identify the same underlying implementation defect:

- Write ONE `Root finding:` block in §3 naming the root cause, listing all affected rule IDs with each rule's own evidence.
- Retain every per-rule verdict in §8 (finding consolidation never erases per-rule results).
- Count the defect once in §2's executive summary; per-rule counts stay per-rule and say so.

Not every co-occurring finding is one root — only defects with a genuinely shared root cause. When in doubt, keep separate.

### 9. Cross-cutting security observations

Per [command step 9a](../.claude/commands/capm-review-milestone.md):

If strong evidence surfaces of a serious security issue whose owning rule belongs to a *later* milestone (e.g., an injection pattern at M3 while `CAP-SEC-013` is M4 PRIMARY):

- Do NOT fail this milestone with the later rule (unless the matrix maps it here — check for CONDITIONAL rows with matching profile flags).
- Do NOT drop it.
- Record it in §5.1: owning rule ID, owning milestone, evidence (file:line), risk, immediate-remediation recommendation, status (new / carried forward).
- Carry it into every subsequent review until the owning milestone evaluates the rule.

### 10. Gate evaluation

Per [matrix §1.3](../development/rule-milestone-matrix.md#13-milestone-gate-results):

- **PASS** — all HARD PASSed, all SOFT resolved or recorded-justified, no NOT ASSESSABLE that would materially affect the gate.
- **PASS WITH EXCEPTIONS** — every HARD FAIL is covered by a verified, in-scope, unexpired [AI-DOC-002 exception record](exception-governance.md); SOFT justifications recorded.
- **FAIL** — at least one uncovered HARD-GATE violation.
- **NOT READY** — required evidence missing at scale (multiple `NOT ASSESSABLE`); the milestone cannot be evaluated as-is.
- **NOT APPLICABLE** — the milestone is out of scope for this project (rare — usually driven by profile).

Never silently downgrade a HARD gate. An expired or out-of-scope exception is rejected with the specific reason in §9.

### 11. Exception verification

Per [exception-governance.md § Review-time verification](exception-governance.md#review-time-verification-what-claude-code-does-at-command-step-8b):

- Load `docs/capm/exceptions/` from the adopting project.
- For each HARD FAIL, look for a record whose `rule` matches.
- Verify approval status, expiry, scope covers the specific evidence, milestone in `milestones_covered`, all required fields present.
- If verified: report §9 lists it; the gate moves toward `PASS WITH EXCEPTIONS`.
- If not verified: rejected with the specific reason; the finding still blocks.

The reviewer never approves an exception, never creates a record, never edits a record. Verification is read-only.

### 12. Remediation plan

Per [remediation-plan template](../templates/remediation-plan-template.md). For FAIL findings only. Advisory — remediation is separate development work performed via `/capm-develop`, not by the reviewer.

Do NOT apply fixes during the review (AI-REVIEW-012). Do NOT propose changes that would weaken the rule.

### 13. Re-review

For `/capm-review-milestone re-review Mx`:

- Load the prior report; scope to its FAILed / NOT-ASSESSABLE rules plus anything the remediation touched.
- **Actually re-run each rule's detection** — never infer resolution from "files changed".
- If `CAP-SEC-001` remediated → **mandatory** `CAP-SEC-011` re-walk. Restriction differentials introduced by the fix may create newly-reachable stricter targets.
- Re-verify CAPire sources — fresh fetches (see § 5 above).
- Recalculate the gate.
- The re-review report scopes clearly — do not extrapolate PASS to rules not re-evaluated (their prior verdicts carry forward, but the scope statement makes that explicit).

### 14. Final traceability

Every finding in the report must trace:

`Rule ID → Project evidence (file:line) → CAPire source → Source status → Verdict`

Report §10 (Standards & CAPire evidence verification) is the canonical table.

A finding that does not trace is a report defect. Fix the report before delivering it.

## Recording quality

- **Findings must be actionable.** Not "security issue found" — "authorization missing on `OrdersService` at `srv/orders-service.cds:3` per `CAP-SEC-001` detection step 3; add service-level `@requires` referencing the existing `Admin`/`User` scopes in `xs-security.json`".
- **Impact stated in operational terms**, especially in `LIVE` reviews — "any authenticated user has full generic CRUD on all exposed orders including internal margin".
- **Remediation refers to existing artifacts** where possible — the compensating control that already exists, the aspect file that already handles related concerns, the ADR pattern the project already uses.

## Boundaries

- The review is bounded to the milestone's evaluated set. Never phrase findings as "the project does not comply with CAP best practices" — the correct scope claim is *"passed the applicable Mx standards evaluated in this review"* (AI-DOC-004).
- ORG-PENDING findings state that blocking authority derives from this standard's own governance, subject to ratification (see [org-ratification.md](org-ratification.md)) — never presented as SAP requirements.
- GEN and AI-REC findings are similarly labeled — never as SAP.
- A single PASS report does not equate to "certified", "SAP compliant", or "production-proven". See [adoption-boundaries.md](adoption-boundaries.md).

## Where to go next

- The review command in detail → [.claude/commands/capm-review-milestone.md](../.claude/commands/capm-review-milestone.md).
- The review model (what a review is) → [reviews/review-model.md](../reviews/review-model.md).
- The AI-REVIEW rules that constrain Claude Code as reviewer → [standards/ai/ai-review-rules.md](../standards/ai/ai-review-rules.md).
- The exception mechanism → [exception-governance.md](exception-governance.md).
- Real-project examples → [pilots/pilot-1-scs-m3-round-3/review-report.md](../pilots/pilot-1-scs-m3-round-3/review-report.md).
