# AI-DOC — Documentation & Decision Rules for Claude Code

Authority: **ORG** (all rules). Runtime: Both. Status: Active.
These rules govern what Claude Code must write down while developing or reviewing.

---

## AI-DOC-001 — Material decisions are recorded
**Severity: High.** Design decisions with lasting consequences (model shape, service cut, integration approach, security design, chosen CAP capability vs alternative) MUST be recorded — as an ADR in the project's decision log or, minimally, in the completion report — including options considered and why the chosen one won.

## AI-DOC-002 — Exceptions are documented at the point of deviation
**Severity: High.** Every approved deviation from a Layer 2 rule MUST be recorded with: rule ID, reason, scope, approver, and (where feasible) a code-adjacent marker/comment referencing the exception, so a later reviewer finds it instead of re-flagging it.

## AI-DOC-003 — Reports follow the canonical templates
**Severity: Medium.** Completion reports use [completion-report-template.md](../../templates/completion-report-template.md); review reports use [review-report-template.md](../../templates/review-report-template.md). Sections are never silently dropped — inapplicable sections say so.

## AI-DOC-004 — Claims carry their authority level
**Severity: High.** In everything Claude writes (docs, ADRs, reports, comments), CAP-technical claims follow [authority levels](../../docs/authority-levels.md): SAP-documented statements carry a reference; unreferenced guidance is labelled as general practice, org policy, or Claude recommendation. No unsupported "SAP requires…" — ever (see AI-REVIEW-006).

## AI-DOC-005 — Documentation changes travel with code changes
**Severity: Medium.** When a change alters behavior, API surface, configuration, or operations, the affected project documentation (README, API docs, deployment notes) MUST be updated in the same increment.

## AI-DOC-006 — Write for the maintainer, not the transcript
**Severity: Low.** Code comments state constraints and non-obvious *why*; they do not narrate what code does, reference the AI session, or justify the change to a reviewer. Documents are self-contained — no reliance on conversation context that a future reader won't have.
