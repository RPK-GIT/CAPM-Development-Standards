# Worked Example — M3 Milestone Review

> **ILLUSTRATIVE / NON-PRODUCTION.**
> Everything in this folder — project, code, findings, report — is **fictional**, created solely to demonstrate the `/capm-review-milestone` workflow. It is **not** a real review, sets **no** precedent, and its contents are **not** part of the normative standard. Do not cite it as evidence for or against any rule.

## What this example demonstrates

A fictional Node.js CAP project ("granary-orders") reviewed at **M3 — Services/API**, showing the full chain:

| Step | Artifact |
|---|---|
| 1. Project profile | [`capm-profile.yaml`](capm-profile.yaml) — machine-readable capability flags |
| 2. Rule filtering | 15 M3-primary rules → **10 evaluated** (profile filters remove DRAFTS/MEDIA/MCP/CONCURRENT-PAGING/INSTANCE-AUTH rules) — shown in the report §1/§7 |
| 3. Evidence collection | Fictional repository files in [`fixture.md`](fixture.md), cited file:line in the report |
| 4. Rule evaluation | One verdict per rule: 7 PASS, **2 FAIL** (one HARD, one SOFT), **1 NOT ASSESSABLE** |
| 5. CAPire verification | Report §10 — per-rule source status (incl. one CURRENT-BUT-EVOLVING illustration) |
| 6. Gate evaluation | HARD-GATE FAIL (CAP-SEC-001) → milestone result **FAIL** |
| 7. Final report | [`review-report.md`](review-report.md) — the complete report per the template |
| 8. Remediation | Report §11 — plan items per the remediation template, no fixes applied |

## How to reproduce the flow

In a real adopting project: place a `capm-profile.yaml` at the root (from [the template](../../templates/capm-profile-template.yaml)), then run `/capm-review-milestone M3`. The command executes the 14-step procedure in [.claude/commands/capm-review-milestone.md](../../.claude/commands/capm-review-milestone.md).
