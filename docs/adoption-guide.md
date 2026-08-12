# Adoption Guide

How a new SAP CAP project adopts CAPM Development Standards v1.0. Practical, ~15 minutes to set up the first project.

Assumptions: you have a CAP project (or are starting one), you have Claude Code available, and your organization has ratified the ORG-PENDING items or accepted that they read as `ORG-PENDING policy findings` per [org-ratification.md](org-ratification.md).

## The ten-step path

### 1 — Make the standards reference available

Choose one:

- **Sibling checkout** — clone `CAPM-Development-Standards` next to your project. Simplest for evaluation.
- **Git submodule** — pin your project to a tagged version (recommended for production adoption).
- **Vendored copy** — copy the `standards/`, `development/`, `reviews/`, `templates/`, `.claude/`, and `docs/` folders into your project. Use only when submodule is not an option; you own re-verification.

Record the pinned version in your project's profile (step 3, field `standard.version`). The v1.0 tag is `v1.0.0`.

### 2 — Copy the profile template

```
<adopting-project>/
  capm-profile.yaml       ← copied from templates/capm-profile-template.yaml
  .claude/commands/
    capm-develop.md       ← copied from .claude/commands/capm-develop.md
    capm-review-milestone.md
```

Templates: [`templates/capm-profile-template.yaml`](../templates/capm-profile-template.yaml), [`.claude/commands/`](../.claude/commands/capm-develop.md). Both command files reference the standards repo at the `standard.version` you set.

### 3 — Fill the profile

Open `capm-profile.yaml` and edit — every field is deliberate.

- `standard.version` — `v1.0.0` (or the pinned commit SHA).
- `project.name` — your project name.
- `project.milestone` — where you are in the lifecycle right now. `M0` for a greenfield start; `LIVE` for retrospective adoption of an already-deployed project. See [project-profile.md § Review modes](../development/project-profile.md#review-modes-milestone-position-vs-reviewed-milestone-added-2026-08-12-round-1-calibration).
- `cap.runtime` — `nodejs` or `java`. Establishable from `package.json` / `pom.xml`.
- `cap.version` — major line (`10`, `5` — see [version-management.md](version-management.md)).
- Everything else — deliberate decisions. The template's comments name the driving rule for each flag.

**Never invent values.** If you don't know a workload flag, set it `unknown` — see [project-profile.md workload evidence states](../development/project-profile.md#workload-flag-evidence-states-added-2026-08-12-round-1-calibration). `unknown` keeps the dependent rule NOT ASSESSABLE (honest); a guessed `false` silently filters it away (dishonest).

**Never put secrets in the profile.** The profile is a capability declaration only. Committing `clientsecret`/`password`/`certificate` there is a CAP-SEC-017 finding.

### 4 — Determine your capability flags accurately

Cross-check each flag against the repository, not against your mental model:

- `capabilities.eventing`: does the project have `cds.requires.messaging` or a message broker config?
- `capabilities.remote_services`: any `.csn`/`.edmx` under `srv/external/`?
- `capabilities.drafts`: any `@odata.draft.enabled` annotation?
- `capabilities.personal_data`: are any user IDs, emails, names, or business-partner records stored/served?
- `capabilities.ui`: any Fiori/UI5 apps under `app/`?

Contradictions between profile and repository stop the review and require you to correct the profile. See [project-profile.md § Validation procedure](../development/project-profile.md#validation-procedure-executed-by-both-commands-before-any-rule-work).

### 5 — Select the milestone you're evaluating

Two common cases:

- **Greenfield DEVELOPMENT** — set `project.milestone: Mx` where `Mx` is the milestone you are actively working on. Review mode = DEVELOPMENT.
- **Retrospective adoption on an existing project** — set `project.milestone: LIVE`; run reviews per milestone starting from M3 (Services/API) or wherever risk is highest. Review mode = RETROSPECTIVE — findings represent current operational risk.

The milestone lifecycle is documented in [development/lifecycle.md](../development/lifecycle.md).

### 6 — Run `/capm-develop` during development

For DEVELOPMENT mode. Claude Code:

- Loads your profile, filters rules by milestone + capability + runtime.
- Reads the applicable rule bodies (never all 134).
- Implements against the rules — prefers CAP-native mechanisms.
- Produces evidence (annotations, tests, config) as it goes.
- Self-validates at completion using the [completion-report template](../templates/completion-report-template.md).

Command: [`.claude/commands/capm-develop.md`](../.claude/commands/capm-develop.md). See also the [developer guide](developer-guide.md).

### 7 — Run `/capm-review-milestone Mx` at milestone completion

Read-only. Claude Code:

- Validates the profile.
- Selects PRIMARY + FINAL-GATE + SUPPORTING rules for the milestone (SUPPORTING selection differs by mode — see [review command step 5](../.claude/commands/capm-review-milestone.md)).
- Collects evidence from the repository (never developer assertions).
- [Verifies CAPire](../reviews/capire-verification.md) sources at report time.
- Applies gate rules.
- Produces the report per the [review-report template](../templates/review-report-template.md).

**The command MUST NOT modify application code.** If it does, that is a workflow violation of AI-REVIEW-012.

### 8 — Review the findings

- **Executive summary** (report §2) counts root findings once, not per rule.
- **Critical rules** (report §3) get their own subsection.
- **Root-finding consolidation** (report §3): multiple rules pointing to the same defect become one block naming the root cause, with all affected rule IDs retained.
- **Cross-cutting security observations** (report §5.1): findings owned by a later milestone's rule, visible here so they are not lost. Carry forward to subsequent reviews until the owning milestone evaluates them.

Every finding traces to: rule → file:line evidence → CAPire source → source status → verdict. If a finding does not, that is a report defect — do not act on it until it does.

### 9 — Remediate

- Prefer CAP-native mechanisms (annotations, aspects) over custom code.
- Address root findings, not per-rule symptoms — one root fix typically flips several rule verdicts.
- Do NOT weaken tests to make findings pass.
- Do NOT suppress the finding.
- For Critical/HARD findings that genuinely cannot be remediated in this milestone, file an [exception record](exception-governance.md) — approver required, expiry required, scope required, compensating control required.

### 10 — Re-review

- Run `/capm-review-milestone re-review Mx`.
- Claude re-runs detection for previously failed/NOT-ASSESSABLE rules + touched files.
- SEC-011 re-walks mandatorily if SEC-001 was remediated (restriction differentials).
- Rules that are actually fixed → PASS. Rules that are not → still FAIL.
- Repeat until the gate PASSes or the exception records are approved.

Then advance `project.milestone` and repeat for the next milestone.

## First-project walkthrough (concise)

Suppose you have a greenfield Node.js CAP project at M3 (services/API), single-tenant, XSUAA, Cloud Foundry, HANA, with 4 Fiori apps and 3 S/4 remote services.

1. `git submodule add https://github.com/RPK-GIT/CAPM-Development-Standards standards-ref`
2. `cp standards-ref/templates/capm-profile-template.yaml capm-profile.yaml`
3. Edit:
   ```yaml
   standard: { version: "v1.0.0" }
   project: { name: "orders", milestone: M3 }
   cap: { runtime: nodejs, version: "10", odata: true }
   tenancy: { multitenant: false, sidecar: null }
   identity: { service: xsuaa, new_project: true }
   capabilities:
     eventing: false
     remote_services: true    # 3 external OData
     mashups: true            # remote entities projected locally
     custom_operations: true
     ui: true
     drafts: true             # if you use @odata.draft.enabled
     personal_data: false     # honest — no user data yet
     # remaining: false
   deployment: { target: cloud-foundry, multi_landscape: false }
   database: { production: hana, legacy_hdbcds: false }
   workload:
     concurrent_edit: unknown  # source: pending owner input
     # rest: false
   ```
4. `mkdir -p .claude/commands && cp standards-ref/.claude/commands/*.md .claude/commands/`
5. In Claude Code: `/capm-develop` — work on services/authorization until self-validation reports clean.
6. `/capm-review-milestone M3` — get the initial report.
7. Read findings, remediate (or record exceptions with approver), re-review.
8. When the gate PASSes: advance to M4.

Expected first-review outcome for a well-modeled greenfield project: a small number of SOFT findings + zero HARD FAILs. If the report is longer, that is honest signal — remediate the roots before proceeding.

## Common first-time issues

- **All 134 rules load** — you are reading the catalog directly instead of running the review command. The command filters; the catalog is the reference.
- **Report says "profile CONTRADICTORY"** — a capability flag contradicts the repository (e.g., `eventing: false` while `cds.requires.messaging` is configured). Correct the profile, not the repository.
- **PERF-001 comes back NOT ASSESSABLE** — you set `concurrent_paging: false` without evidence. Either establish evidence, or set `unknown` and get owner input; do not guess.
- **CAPire source flagged STALE** — the [quarterly cadence](re-verification-cadence.md) is behind. Bump `Last verified` after re-fetching per the [CAPire protocol](../reviews/capire-verification.md).

## Next steps

- **Developer perspective** — [developer-guide.md](developer-guide.md).
- **Reviewer perspective** — [reviewer-guide.md](reviewer-guide.md).
- **Management perspective** — [executive-overview.md](executive-overview.md).
- **What v1.0 does NOT guarantee** — [adoption-boundaries.md](adoption-boundaries.md).
