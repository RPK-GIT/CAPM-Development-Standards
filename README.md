# CAPM Development Standards

**Version 1.0.0** — a production-grade **SAP Cloud Application Programming Model (CAP) Engineering Standard and AI-Assisted Development Playbook**.

This repository is the authoritative, version-controlled source of truth for how our organization designs, develops, tests, secures, deploys, operates — and **reviews** — CAP applications. It is written to be used by **human developers** and by **Claude Code**, both as an implementation assistant and as an independent reviewer at development milestones.

> **Status: v1.0.0 released 2026-08-12.**
> Validated through controlled real-project pilot and remediation experiments (Pilot 1 · Rounds 1–3 on a live BTP project). See [CHANGELOG.md](CHANGELOG.md) and the [executive overview](docs/executive-overview.md).

## What this repository is — and is not

It **is** an engineering standard: a layered system of principles, formally identified rules with evidence and detection guidance, a milestone-based lifecycle, and machine-followable procedures for AI-assisted development and review.

It is **not** a CAP tutorial, a generic style guide, a loose collection of best practices, a prompt library, or a mere checklist. For learning CAP, use the official documentation: <https://cap.cloud.sap/docs/>.

It is **not** SAP-issued or SAP-endorsed. Compliance with this standard is not equivalent to SAP compliance. See [docs/adoption-boundaries.md](docs/adoption-boundaries.md).

## What v1.0 contains

- **134 normative Layer 2 rules** across 20 categories — 40 SAP-REQ · 89 SAP-REC · 2 GEN · 3 ORG-PENDING · 11 Critical (all HARD-gate).
- **M0–M9 lifecycle** with per-milestone checklists and a canonical [rule-milestone matrix](development/rule-milestone-matrix.md) (34 HARD · 94 SOFT · 6 ADVISORY).
- **Claude Code development/review workflow** — [`/capm-develop`](.claude/commands/capm-develop.md) and [`/capm-review-milestone Mx`](.claude/commands/capm-review-milestone.md).
- **CAPire evidence verification** ([reviews/capire-verification.md](reviews/capire-verification.md)) — three verification levels, six source statuses, mandatory traceability chain from rule → project evidence → CAPire source → source status → verdict.
- **Exception governance** ([docs/exception-governance.md](docs/exception-governance.md)) — auditable HARD-gate exception records with approver, expiry, scope, and compensating control.
- **Real-project pilot validation** — three-round closed-loop demonstration on a live BTP application ([pilots/](pilots/pilot-1-scs-m3/README.md)).
- **48 operational validation scenarios** — 40 pre-calibration base + 8 added at Round 1 calibration, all executed under Node.js and Java fixtures + real-project pilot inputs ([docs/operational-validation.md](docs/operational-validation.md), [validation-results-2026-08.md](docs/validation-results-2026-08.md)).

## The three layers

| Layer | What it contains | Where |
|---|---|---|
| **1 — Engineering Principles** | Fundamental CAP engineering principles, grounded in SAP documentation | [`standards/principles/`](standards/principles/engineering-principles.md) |
| **2 — Engineering Rules** | Formal rule catalog (`CAP-ARCH-001`, `CAP-SEC-001`, …) with severity, authority, runtime & version applicability, evidence, detection guidance, examples | [`standards/rules/`](standards/rules/README.md) |
| **3 — AI Development & Review Rules** | Rules governing Claude Code itself (`AI-DEV-…`, `AI-REVIEW-…`, `AI-TEST-…`, `AI-DOC-…`) | [`standards/ai/`](standards/ai/README.md) |

Design: [docs/standard-architecture.md](docs/standard-architecture.md).

## Repository map

```
├── README.md                     ← you are here
├── CHANGELOG.md                  ← v1.0 release + phase-by-phase history
├── CLAUDE.md                     ← how Claude Code operates in and with this repo
├── docs/                         ← the design + governance of the standard
│   ├── standard-architecture.md    ← three-layer architecture, document map
│   ├── authority-levels.md         ← SAP-REQ / SAP-REC / GEN / ORG / AI-REC taxonomy
│   ├── version-management.md       ← CAP / Node.js / Java / DB version policy + register
│   ├── operational-validation.md   ← 48 executable scenarios
│   ├── validation-results-2026-08.md ← v1.0 validation execution record
│   ├── rule-governance.md          ← rule lifecycle (propose → amend → deprecate)
│   ├── exception-governance.md     ← HARD-gate exception records — approver, expiry, scope
│   ├── org-ratification.md         ← three ORG-PENDING rules awaiting organizational ratification
│   ├── re-verification-cadence.md  ← quarterly CAPire sweep, release triggers, annual review
│   ├── adoption-guide.md           ← ten-step adoption path for a new project
│   ├── developer-guide.md          ← practical guide for developers
│   ├── reviewer-guide.md           ← practical guide for reviewers
│   ├── executive-overview.md       ← 2–3-page management-facing overview
│   └── adoption-boundaries.md      ← what v1.0 does NOT guarantee (read this)
├── standards/
│   ├── principles/               ← Layer 1: engineering principles
│   ├── rules/                    ← Layer 2: formal rule catalog by category
│   └── ai/                       ← Layer 3: AI development & review rules
├── development/
│   ├── lifecycle.md              ← M0–M9 lifecycle
│   ├── development-model.md      ← how Claude Code implements against this standard
│   ├── project-profile.md        ← capm-profile.yaml schema + validation
│   ├── rule-milestone-matrix.md  ← rule↔milestone mapping, gates, conditions
│   └── milestones/               ← operational checklists M0–M9
├── reviews/
│   ├── review-model.md           ← how Claude Code reviews projects against this standard
│   └── capire-verification.md    ← live SAP-source verification at report time
├── .claude/commands/             ← /capm-develop and /capm-review-milestone
├── examples/                     ← worked examples (ILLUSTRATIVE / NON-PRODUCTION)
│   ├── worked-example-m3/          ← Node.js
│   └── worked-example-m3-java/     ← Java
├── pilots/                       ← real-project pilot record — historical, preserved
│   ├── pilot-1-scs-m3/             ← Round 1 (initial live review)
│   ├── round-1-calibration.md      ← Round 1 calibration decisions
│   ├── pilot-1-scs-m3-round-2/     ← Round 2 (re-review, no remediation)
│   └── pilot-1-scs-m3-round-3/     ← Round 3 (real remediation → FAIL→PASS)
├── references/
│   ├── sap-cap-sources.md          ← source map of official SAP CAP documentation
│   ├── research-gaps.md            ← where SAP guidance ends and org policy must begin
│   ├── candidate-rules.md          ← researched candidate rule inventory (input to Layer 2)
│   └── candidate-dispositions.md   ← what happened to each candidate per authored batch
└── templates/
    ├── capm-profile-template.yaml
    ├── rule-template.md
    ├── review-report-template.md
    ├── remediation-plan-template.md
    ├── completion-report-template.md
    └── milestone-checklist-template.md
```

## Authority model

Every statement in this standard carries an explicit authority level so that SAP's guidance is never confused with our own policy:

1. **SAP-REQ** — SAP-documented requirement
2. **SAP-REC** — SAP-documented recommendation
3. **GEN** — general engineering practice
4. **ORG** — organization-specific standard (`ORG-PENDING` until ratified — see [org-ratification.md](docs/org-ratification.md))
5. **AI-REC** — Claude recommendation

Definitions and usage rules: [docs/authority-levels.md](docs/authority-levels.md).

## Core working principle

**CAP-native solution first.** Where CAP provides an established capability (generic CRUD handlers, CDS-based validation, authorization annotations, messaging, MTX, `cds.test`, …), it is used in preference to custom abstractions, unless a documented reason exists not to. This principle is grounded in SAP's own best-practices guidance and runs through every layer of this standard.

## Using this standard

- **To adopt:** [docs/adoption-guide.md](docs/adoption-guide.md) — ten steps, ~15 minutes for the first project.
- **To develop:** [`/capm-develop`](.claude/commands/capm-develop.md) — binds [development-model.md](development/development-model.md) + the [matrix](development/rule-milestone-matrix.md) to your profile. See also [developer-guide.md](docs/developer-guide.md).
- **To review:** [`/capm-review-milestone <Mx>`](.claude/commands/capm-review-milestone.md) — profile-driven, matrix-filtered, [CAPire-verified](reviews/capire-verification.md); read-only. See also [reviewer-guide.md](docs/reviewer-guide.md).
- **To understand at a glance:** [docs/executive-overview.md](docs/executive-overview.md) (2–3 pages).
- **To evolve the standard:** rules are modular Markdown files with stable IDs; propose changes per [rule-governance.md](docs/rule-governance.md).

## Roadmap and validation history

Full phase-by-phase history: [CHANGELOG.md](CHANGELOG.md). Summary at v1.0:

| Phase | Deliverable | Status |
|---|---|---|
| 1 — Research & Foundation | Repo architecture, source map, candidate rules, gaps, lifecycle & review/development models, AI rule family | ✅ complete |
| 2 — Rule Catalog | 134 rules across 20 categories | ✅ complete |
| 3 — Lifecycle Integration | Matrix, checklists, commands, profile, worked examples; 40 → 48 validation scenarios (historical Round-1-calibration record notes this correction) | ✅ complete |
| 4 — Pilot & Calibration | Real-project Pilot 1 (Rounds 1–3), Round 1 calibration, real remediation experiment | ✅ complete — closed-loop demonstrated ([Round 3 report](pilots/pilot-1-scs-m3-round-3/review-report.md)) |
| 5 — v1.0 Finalization | Governance + adoption package, README/CLAUDE.md finalization, CHANGELOG, `v1.0.0` tag | ✅ this release |
| Adoption (post-v1.0) | Organizational ratification of ORG-PENDING rules; broader project pilots; annual governance review | ⬜ organizational |

## What v1.0 does NOT claim

- Not SAP-certified. Not SAP-approved. Not production-proven.
- A PASS milestone gate is not a guarantee of defect-free software.
- Not a replacement for SAP documentation, security testing, runtime testing, architecture review, or organizational policy.

Full boundaries: [docs/adoption-boundaries.md](docs/adoption-boundaries.md).

## Primary authority

The primary authoritative source for all CAP-technical content is the official SAP CAP documentation: **<https://cap.cloud.sap/docs/>**, supplemented by official SAP Help Portal pages and official CAP release notes/changelogs. Third-party material is never treated as authoritative where SAP documentation exists. All source evidence is recorded in [references/sap-cap-sources.md](references/sap-cap-sources.md); the [re-verification cadence](docs/re-verification-cadence.md) keeps it current.
