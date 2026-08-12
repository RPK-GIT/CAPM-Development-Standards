# CLAUDE.md — Operating Instructions for Claude Code

This repository is the **CAPM Engineering Standard**. Claude Code interacts with it in three distinct modes. Determine the mode first; it changes what you are allowed to do.

## Mode 1 — Maintaining this standard (working *in* this repo)

- This repo contains **documentation only**. Never add application code, build tooling, or dependencies here.
- Every CAP-technical claim must carry an authority level per [docs/authority-levels.md](docs/authority-levels.md) and a source entry in [references/sap-cap-sources.md](references/sap-cap-sources.md). **Never fabricate SAP guidance.** If you cannot cite an official SAP page, the statement is at most `GEN`, `ORG`, or `AI-REC` — never `SAP-REQ`/`SAP-REC`.
- Rules are modular files with **stable IDs** (see [standards/rules/README.md](standards/rules/README.md)). Never renumber or reuse a retired ID; mark rules `Deprecated` instead of deleting them.
- New rules must use [templates/rule-template.md](templates/rule-template.md) with every field populated, including runtime applicability (Node.js / Java / Both) and CAP version applicability.
- CAP guidance is version-sensitive. When touching any rule, verify the cited SAP page still says what the rule claims; record the verification date in the rule's `Last verified` field.
- Keep SAP source evidence (`references/`) separate from organization policy (`standards/`). Gaps in SAP guidance are recorded in [references/research-gaps.md](references/research-gaps.md), not papered over.
- Verify relative links after moving or renaming files.
- Never commit secrets, credentials, `.env` files, or tokens.

## Mode 2 — Developing a CAP application against this standard

Follow [development/development-model.md](development/development-model.md) and the Layer 3 rules in [standards/ai/](standards/ai/README.md). In brief:

1. Read the applicable standards for the current milestone ([development/lifecycle.md](development/lifecycle.md)).
2. Inspect the existing implementation before proposing anything.
3. Prefer **CAP-native capabilities** over custom code (see `AI-DEV-004`).
4. Implement incrementally; add or update tests with the code.
5. Self-validate against applicable `CAP-*` rules; produce a completion report ([templates/completion-report-template.md](templates/completion-report-template.md)).

**Never without explicit approval:** major architectural changes, new frameworks/abstractions duplicating CAP capabilities, bypassing CAP abstractions, weakening security, removing or disabling tests to make them pass, silently ignoring a standard, or claiming compliance without evidence. Full list: `AI-DEV-010` … `AI-DEV-016`.

## Mode 3 — Reviewing a CAP application against this standard

When asked to *“Review this project against the CAPM Engineering Standard”* (or similar), follow [reviews/review-model.md](reviews/review-model.md) and the `AI-REVIEW-*` rules exactly:

- Inspect the **actual repository**; every finding must cite a rule ID and, where possible, file/line evidence.
- Verdicts are `PASS` / `FAIL` / `NOT APPLICABLE` / `NOT ASSESSABLE` per rule — never impressionistic.
- Distinguish **defects** (rule violations) from **recommendations**.
- Never claim SAP requires something without a supporting reference in the rule or source map.
- Output the report using [templates/review-report-template.md](templates/review-report-template.md).

## Quick navigation

| Need | Read |
|---|---|
| The standard's design | [docs/standard-architecture.md](docs/standard-architecture.md) |
| Authority taxonomy | [docs/authority-levels.md](docs/authority-levels.md) |
| Version policy | [docs/version-management.md](docs/version-management.md) |
| Principles (Layer 1) | [standards/principles/engineering-principles.md](standards/principles/engineering-principles.md) |
| Rule catalog (Layer 2) | [standards/rules/README.md](standards/rules/README.md) |
| AI rules (Layer 3) | [standards/ai/README.md](standards/ai/README.md) |
| Milestones & gates | [development/lifecycle.md](development/lifecycle.md) · [rule-milestone matrix](development/rule-milestone-matrix.md) · [per-milestone checklists](development/milestones/m0-requirements.md) |
| SAP source map | [references/sap-cap-sources.md](references/sap-cap-sources.md) |
