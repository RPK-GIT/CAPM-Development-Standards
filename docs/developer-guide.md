# Developer Guide

Practical usage of CAPM Development Standards v1.0 for developers building CAP applications. Keep this next to [/capm-develop](../.claude/commands/capm-develop.md) while working.

The goal is to make each milestone gate PASS the first time by working within the standard as you write code — not to satisfy the reviewer post hoc.

## What you're expected to do

- **Own the profile.** `capm-profile.yaml` at your project root declares your project's shape. Keep it honest: workload flags are `unknown` where you can't establish them, not `false`. See [adoption-guide.md § 4](adoption-guide.md#4--determine-your-capability-flags-accurately).
- **Load only what applies.** Never scan the whole 134-rule catalog. `/capm-develop` filters by profile + milestone; the catalog remains a reference you drill into when a rule appears in output.
- **Produce evidence as you code.** Annotations, tests, configuration — the artifacts the rule's *Evidence expected in code* section names. Not commit messages; not verbal assertions.
- **Handle HARD constraints deliberately.** HARD-gate rules block the milestone unless you have an approved [exception](exception-governance.md). Design for them, do not defer them.
- **Self-validate before requesting review.** The [completion report template](../templates/completion-report-template.md) is the checklist Claude uses under `/capm-develop`; use it the same way a human would.

## Using `/capm-develop`

Command: [.claude/commands/capm-develop.md](../.claude/commands/capm-develop.md). Common flow:

1. **State the milestone and task**, e.g., *"I'm at M3, implementing the OrdersService authorization model."*
2. **Claude loads the profile + milestone rules**, filters by capability/runtime, and reads only the applicable rule bodies.
3. **Claude inspects the current implementation** before proposing anything (AI-DEV-001).
4. **Claude prefers CAP-native mechanisms** (AI-DEV-004): `@requires` before handler-based auth; `@assert.*` before imperative validation; generic providers before hand-written CRUD; `managed` before manual timestamps.
5. **Claude adds/updates tests** with the code (AI-DEV-013). If tests fail, the defect gets fixed — not the test (AI-DEV-014).
6. **Claude self-validates** and emits a completion report with per-rule evidence pointers, ready for `/capm-review-milestone`.

If Claude hits an architectural boundary (service split, framework change, security-weakening change), it STOPs and produces an *architectural-deviation report* (AI-DEV-010) rather than proceeding silently.

## How standards are selected

At the moment `/capm-develop` starts, the selected rule set is:

```
selected = (milestone.PRIMARY ∪ milestone.FINAL-GATE)
         + (relevant SUPPORTING per review-mode logic)
         - runtime-filtered  (Java-only rules absent in Node.js projects, etc.)
         - capability-filtered (rules gated by profile flags off)
         - version-filtered  (rules not applicable at your CAP version)
```

Typical size ranges by milestone: M0 3-6 · M1 12-15 · M2 15-18 · M3 8-13 · M4 20-29 · M5 8-16 · M6 12-14 · M7 5-7 · M8 12-15 · M9 4-8. Full mapping: [rule-milestone-matrix.md](../development/rule-milestone-matrix.md).

You never see all 134 in one working session.

## HARD / SOFT / ADVISORY — what they mean in practice

| Gate | If it FAILs | What you should do |
|---|---|---|
| **HARD** | The milestone cannot PASS. There is no soft option. | Fix it in this milestone. If genuinely impossible, file an [exception record](exception-governance.md) with an approver and expiry. Design work should not push HARD-gate concerns into later milestones. |
| **SOFT** | The milestone can proceed only with **explicit recorded justification** in the milestone checklist. | Fix it, or record why not with the alternative you chose and a review date. Recording SOFT justifications is lighter than exception records — one paragraph in the checklist, not an approver signature. |
| **ADVISORY** | Never blocking. | Read it; apply the guidance where cheap; ignore where it doesn't fit. |

All 11 Critical rules are HARD. Never treat a Critical finding as "we'll get to it".

## Producing evidence

Evidence is what the reviewer collects (file:line) or what the framework detects (a CAP compile inspection). Ways to make evidence obvious:

- **Annotations in the model.** `@requires`, `@restrict`, `@assert.*`, `@readonly`, `@odata.etag`, `@protocol` — these ARE the evidence for many rules. Prefer them over handler code that duplicates them.
- **Aspect files.** Put authorization in `srv/access-control.cds`, UI annotations in `app/services.cds` / per-app annotation files — reviewers find them by name.
- **Config values.** `application.yaml` (Java) or `.cdsrc.json` / `package.json` `cds:` block (Node.js) with the property the rule expects.
- **Tests with cited paths.** `cds.test` (Node.js) or MockMvc (Java) files at the paths the milestone checklist names.
- **ADRs / decision records.** Where a rule permits a recorded decision (e.g., `CAP-SRV-008`'s last-write-wins path). Keep them in `docs/adr/` in your project — the reviewer reads there.

Developer assertions are not evidence unless the rule explicitly accepts a documented decision. See [AI-REVIEW-010](../standards/ai/ai-review-rules.md).

## When a finding lands on your commit

Order of operations:

1. **Read the rule** the finding cites — the full body (statement / rationale / detection guidance / good/bad examples). The catalog is not decoration; it explains the correct fix.
2. **Identify the root cause**, not just the specific line. If two rules point to the same defect (finding-consolidation convention), fixing the root usually flips both. Root-finding blocks in the review report §3 are already grouped for you.
3. **Prefer the declarative fix.** If the rule's *Good example* uses an annotation, that is the intended shape of the fix. Handler code duplicating an annotation is the anti-pattern the rule flags.
4. **Do not delete or weaken the test that surfaced it** (AI-DEV-014). If a test needs to change because behavior legitimately changes, update the test to cover the new behavior — don't remove coverage.
5. **Do not silently ignore** (AI-DEV-016). If a rule truly does not apply in your context, work out why and either update the profile (if a capability changed) or record the situation as a recommendation observation — not as a suppressed finding.

## How to request an exception

Not for every SOFT finding — SOFT items only need a recorded justification in the milestone checklist. Exceptions are for **HARD** findings genuinely unavoidable in this milestone.

1. Create `docs/capm/exceptions/<YYYY-MM-DD>-<rule-id>-<slug>.md` in your project.
2. Fill every required field from [exception-governance.md § Exception record](exception-governance.md#exception-record--required-fields) — including approver, expiry, scope, compensating control, re-review trigger. Missing any field = record does not count.
3. Get the required approver(s) — for Critical rules, two approvers.
4. Re-run `/capm-review-milestone Mx`. The report §9 lists the record; if verified, the gate moves to `PASS WITH EXCEPTIONS`; if not verified, the specific reason is reported.
5. The finding still appears in the report — exceptions do not delete findings, they document accepted risk.

## Re-review

After remediation, run `/capm-review-milestone re-review Mx`. Claude:

- Loads the prior report.
- Scopes to previously FAILed / NOT-ASSESSABLE rules + files your remediation touched.
- **Actually re-runs each rule's detection** — never infers a fix from "files changed".
- If `CAP-SEC-001` was remediated, the mandatory `CAP-SEC-011` re-walk runs — because restriction differentials may have been introduced (see [Pilot 1 Round 3](../pilots/pilot-1-scs-m3-round-3/review-report.md) for a worked example).
- Re-verifies CAPire sources (not silently reused from the prior report).
- Recalculates the gate.

If a finding is genuinely fixed, it PASSes. If it isn't, it stays FAIL. There is no in-between.

## Common developer questions

- **"The rule says use `@restrict`, but my handler already checks ownership. Why does the finding still fire?"** Because handler filters for entities *lacking a model restriction* are the specific detection step. Add the `@restrict … where …` and either remove the handler or leave it (belt-and-suspenders); the model is the source of truth.
- **"The report says CAPire verified 2026-08-12 but I haven't touched anything since; do I need to re-verify?"** Not per review; the [quarterly cadence](re-verification-cadence.md) or a load-bearing source change triggers re-verification. Individual reviews re-fetch what's material to the evaluated rules.
- **"A rule cites a CAP version I'm not on."** The [version register](version-management.md) is the source of truth; the rule's *CAP version* field names the range it applies to. If your version is outside the range, the rule is not applicable (report §7).
- **"The report is long. Can we shorten rules?"** No. Reports are long when there is a lot to report. Fix the roots and the report shortens by itself. See [Pilot 1 Round 3](../pilots/pilot-1-scs-m3-round-3/review-report.md) where two remediations closed one Critical HARD + one High SOFT + moved SRV-003 partially.
- **"Can I skip a rule that seems off?"** No — but you can propose a [rule change](rule-governance.md) with evidence. That is a maintainer path, not an in-project workaround.

## Where to go next

- Adoption path (first-time setup) → [adoption-guide.md](adoption-guide.md).
- Reviewer's view (understanding what a reviewer will do) → [reviewer-guide.md](reviewer-guide.md).
- What the standard does NOT guarantee → [adoption-boundaries.md](adoption-boundaries.md).
- The rule catalog itself → [../standards/rules/README.md](../standards/rules/README.md).
