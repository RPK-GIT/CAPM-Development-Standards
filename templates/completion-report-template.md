# Completion Report Template

Produced at the end of every development increment/session ([development model](../development/development-model.md), step 10). It documents what was done and with what evidence — it is **input to** the independent milestone review, never a replacement for it (AI-REVIEW-009).

```markdown
# Development Completion Report

| Field | Value |
|---|---|
| **Project** | name / repository |
| **Branch / revision** | branch, resulting commit SHA(s) |
| **Date** | YYYY-MM-DD |
| **Implementer** | Claude Code (model/version) / human |
| **Milestone** | Mx — name (per development/lifecycle.md) |
| **Standard version** | CAPM Engineering Standard @ <commit/tag> |

## 1. Requirement & understanding
The requirement as implemented, restated (AI-DEV-003). Acceptance criteria addressed.

## 2. Assumptions & open questions
Explicit assumptions made where requirements were ambiguous; questions needing an answer.

## 3. Standards consulted
Layer 2 categories / documents read before implementing (AI-DEV-001).

## 4. CAP capabilities used
Requirement part → CAP-native capability used (with doc reference), per AI-DEV-004.
Residual custom code and why CAP-native means did not cover it.

## 5. Changes made
File-level summary of production changes; notable design points. Architecture impact:
"none — follows existing patterns" or reference to the approved proposal (AI-DEV-010).

## 6. Tests
Tests added/changed and what behavior each covers (incl. unhappy paths, AI-TEST-004).
**Actual execution results** — command(s) run and outcome (AI-TEST-006). Known coverage
gaps stated honestly (AI-TEST-007).

## 7. Self-validation against the standard (AI-DEV-008)

| Rule ID | Result (met / not met / N/A) | Evidence |
|---|---|---|
| CAP-XXX-000 | met | file:line / test name / config |

## 8. Deviations & exceptions
Any rule not complied with: rule ID, reason, approval status (AI-DEV-015, AI-DOC-002).
"None" if none.

## 9. Decisions recorded
ADRs created/updated, or inline record of material decisions (AI-DOC-001).

## 10. Security note
Security-relevant aspects of the change, or "no security-relevant changes" (AI-DEV-013).

## 11. Open items & recommended next steps
Unfinished work, follow-ups, recommendations (labelled AI-REC).
```
