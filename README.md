# CAPM Engineering Standard

A production-grade **SAP Cloud Application Programming Model (CAP) Engineering Standard and AI-Assisted Development Playbook**.

This repository is the authoritative, version-controlled source of truth for how our organization designs, develops, tests, secures, deploys, operates — and **reviews** — CAP applications. It is written to be used by **human developers** and by **Claude Code**, both as an implementation assistant and as an independent reviewer at development milestones.

> **Status: Phase 2 complete (2026-08-12) — the full Layer 2 rule catalog is authored and normative: 134 rules across all 20 categories.**
> Next: Phase 3 — per-milestone rule mappings, checklists, and worked examples. See [Roadmap](#roadmap).

## What this repository is — and is not

It **is** an engineering standard: a layered system of principles, formally identified rules with evidence and detection guidance, a milestone-based lifecycle, and machine-followable procedures for AI-assisted development and review.

It is **not** a CAP tutorial, a generic style guide, a loose collection of best practices, a prompt library, or a mere checklist. For learning CAP, use the official documentation: <https://cap.cloud.sap/docs/>.

## The three layers

| Layer | What it contains | Where |
|---|---|---|
| **1 — Engineering Principles** | Fundamental CAP engineering principles, grounded in SAP documentation | [`standards/principles/`](standards/principles/engineering-principles.md) |
| **2 — Engineering Rules** | Formal rule catalog (`CAP-ARCH-001`, `CAP-SEC-001`, …) with severity, authority, runtime & version applicability, evidence, detection guidance, examples | [`standards/rules/`](standards/rules/README.md) |
| **3 — AI Development & Review Rules** | Rules governing Claude Code itself (`AI-DEV-…`, `AI-REVIEW-…`, `AI-TEST-…`, `AI-DOC-…`) | [`standards/ai/`](standards/ai/README.md) |

See [docs/standard-architecture.md](docs/standard-architecture.md) for the full design.

## Repository map

```
├── README.md                  ← you are here
├── CLAUDE.md                  ← how Claude Code must operate in and with this repo
├── docs/                      ← the design of the standard itself
│   ├── standard-architecture.md   ← three-layer architecture, document map
│   ├── authority-levels.md        ← SAP requirement vs recommendation vs org policy taxonomy
│   └── version-management.md      ← CAP / Node.js / Java / DB version policy
├── standards/
│   ├── principles/            ← Layer 1: engineering principles
│   ├── rules/                 ← Layer 2: formal rule catalog by category
│   └── ai/                    ← Layer 3: AI development & review rules
├── development/
│   ├── lifecycle.md           ← milestone-based development lifecycle (M0–M9)
│   ├── development-model.md   ← how Claude Code implements against this standard
│   ├── rule-milestone-matrix.md ← canonical rule↔milestone mapping, gates, conditions
│   └── milestones/            ← operational checklists M0–M9
├── reviews/
│   └── review-model.md        ← how Claude Code reviews projects against this standard
├── references/
│   ├── sap-cap-sources.md     ← source map of official SAP CAP documentation
│   ├── research-gaps.md       ← where SAP guidance ends and org policy must begin
│   ├── candidate-rules.md     ← researched candidate rule inventory (input to Layer 2)
│   └── candidate-dispositions.md ← what happened to each candidate per authored batch
└── templates/
    ├── rule-template.md
    ├── review-report-template.md
    ├── completion-report-template.md
    └── milestone-checklist-template.md
```

## Authority model

Every statement in this standard carries an explicit authority level so that SAP's guidance is never confused with our own policy:

1. **SAP-REQ** — SAP-documented requirement
2. **SAP-REC** — SAP-documented recommendation
3. **GEN** — general engineering practice
4. **ORG** — organization-specific standard
5. **AI-REC** — Claude recommendation

Definitions and usage rules: [docs/authority-levels.md](docs/authority-levels.md).

## Core working principle

**CAP-native solution first.** Where CAP provides an established capability (generic CRUD handlers, CDS-based validation, authorization annotations, messaging, MTX, `cds.test`, …), it is used in preference to custom abstractions, unless a documented reason exists not to. This principle is grounded in SAP's own best-practices guidance and runs through every layer of this standard.

## Using this standard

- **To develop:** follow [development/development-model.md](development/development-model.md) and the milestone lifecycle in [development/lifecycle.md](development/lifecycle.md).
- **To review:** ask Claude Code: *“Review this project against the CAPM Engineering Standard.”* The procedure and report format are defined in [reviews/review-model.md](reviews/review-model.md).
- **To evolve the standard:** rules are modular Markdown files with stable IDs; propose changes per rule file, never by rewriting the catalog wholesale.

## Roadmap

| Phase | Deliverable | Status |
|---|---|---|
| 1 — Research & Foundation | Repo architecture, source map, candidate rules, gaps, lifecycle & review/development models, AI rule family | ✅ this commit |
| 2 — Rule Catalog | Author full Layer 2 rule catalog from [candidate rules](references/candidate-rules.md) using the [rule template](templates/rule-template.md) | ✅ **complete (2026-08-12)** — 134 rules across all 20 categories; [dispositions](references/candidate-dispositions.md) |
| 3 — Lifecycle Integration | Per-milestone rule mappings, checklists, worked examples | 🔶 in progress — [rule-milestone matrix](development/rule-milestone-matrix.md) + [M0–M9 checklists](development/milestones/m0-requirements.md) done; worked examples & review/development commands next |
| 4 — Pilot & Calibration | Run reviews against real projects, calibrate severities, close gaps in [research-gaps.md](references/research-gaps.md) | ⬜ |

## Primary authority

The primary authoritative source for all CAP-technical content is the official SAP CAP documentation: **<https://cap.cloud.sap/docs/>**, supplemented by official SAP Help Portal pages and official CAP release notes/changelogs. Third-party material is never treated as authoritative where SAP documentation exists. All source evidence is recorded in [references/sap-cap-sources.md](references/sap-cap-sources.md).
