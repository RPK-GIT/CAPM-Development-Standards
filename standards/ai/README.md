# Layer 3 — AI Development & Review Rules

Rules governing how **Claude Code** (or any comparable AI assistant) works on CAP applications under this standard. All Layer 3 rules are **`ORG`** authority — they are our policy for AI usage, not SAP guidance — and apply to **both runtimes** and **all CAP versions** unless stated otherwise.

Unlike Layer 2 (which describes the *application*), Layer 3 describes the *process*: the AI's obligations when inspecting, designing, implementing, testing, validating, reviewing, reporting, and documenting.

| Family | Scope | File |
|---|---|---|
| `AI-DEV` | Development behavior: inspection, design, implementation, guardrails | [ai-development-rules.md](ai-development-rules.md) |
| `AI-REVIEW` | Review behavior: evidence, verdicts, severity, reporting | [ai-review-rules.md](ai-review-rules.md) |
| `AI-TEST` | Test authorship and validation behavior | [ai-testing-rules.md](ai-testing-rules.md) |
| `AI-DOC` | Documentation and decision-recording behavior | [ai-documentation-rules.md](ai-documentation-rules.md) |

These rules are operationalized by two procedures:

- [development/development-model.md](../../development/development-model.md) — the session flow for implementation work
- [reviews/review-model.md](../../reviews/review-model.md) — the procedure for standard reviews

Layer 3 rules use the same [rule template](../../templates/rule-template.md) fields in compact form (severity, statement, rationale, detection) since fields like "CAP version applicability" rarely vary here.
