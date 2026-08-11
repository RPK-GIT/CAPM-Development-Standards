# Version Management

CAP changes monthly; guidance that was correct last year may be wrong today. This document defines (a) how this standard handles version sensitivity, and (b) the currently verified version baseline. **Facts below carry the date they were verified against official SAP pages; re-verify before relying on them.**

## Policy

1. **No unverified "always use X".** Version recommendations appear in this standard only with an official SAP source and a verification date. Where SAP is silent, the entry is marked as an [org-policy gap](../references/research-gaps.md).
2. **Rules carry version applicability.** Every Layer 2 rule states the CAP version range it was verified for (template field `CAP version`). Rules invalidated by a release are updated or deprecated — never silently left stale.
3. **Version-sensitive rule register.** Rules whose substance depends on versions (defaults that flipped, packages that changed, deprecated formats) are flagged in their category files; the register below tracks the themes.
4. **Re-verification cadence (ORG).** The source map and this baseline are re-verified against <https://cap.cloud.sap/docs/releases/> at least quarterly and at every CAP major release. The January 2026 documentation restructuring (many URL moves) shows why: cited URLs rot.
5. **Follow SAP's maintenance model (SAP-REC).** Per the official [release schedule](https://cap.cloud.sap/docs/releases/schedule): monthly minor releases; majors move prior majors to *Maintenance* (≤ 12 months, critical fixes only), then *End of life* (no fixes). Projects must plan major upgrades inside that window; SAP advises consuming latest minors monthly and patches ASAP (CAP Java versions page).

## Verified version baseline — as of 2026-08-11

Source: official release notes/changelog (<https://cap.cloud.sap/docs/releases/2026/jun26>, <https://cap.cloud.sap/docs/releases/2026/changelog>), CAP Java versions page (<https://cap.cloud.sap/docs/java/versions>). Authority: **SAP-REQ/SAP-REC as noted**.

| Component | Current (verified 2026-08-11) | Notes |
|---|---|---|
| `@sap/cds` (Node.js) | **10.x** (10.0.2 seen) | June 2026 major. Do not mix major lines: pairs with `@sap/cds-dk` 10.x, `@sap/cds-compiler` 7.x, `@sap/cds-mtxs` 4.x, `@cap-js/*` 3.x (SAP-REQ) |
| CAP Java (`com.sap.cds`) | **5.x** (5.0.0) active; **4.9.x** maintenance (patches only) | Java 5: Spring Boot ≥ 4, Spring Security 7, Maven ≥ 3.9.14 (SAP-REQ) |
| Node.js | **≥ 22 required**; 24/26 recommended | Node 20 dropped with cds 10 (Node 20 EOL 2026-04). `node:sqlite` default driver needs ≥ 22.5 (SAP-REQ) |
| JDK | **≥ 21 required**; 25 recommended (SapMachine) | JDK 17 dropped with CAP Java 5; SapMachine 17 EOL Sept 2026; SAP Java Buildpack defaults to 21 since 2026-04-02 (SAP-REQ/REC) |
| Databases — production | SAP HANA Cloud (PostgreSQL for edge cases) (SAP-REC) | Deploy format `hdbtable`; `hdbcds` unusable since cds 9 (SAP-REQ) |
| Databases — development | SQLite via `@cap-js/sqlite` (Node), H2 **2.3.x only** (Java) (SAP-REQ for H2 pin) | Old built-in DB services replaced by `@cap-js/*` since cds 8; remove direct `hdb`/`sqlite3` deps (SAP-REC) |
| Express (Node) | `express^5` default when unpinned since Jan 2026 | Flagged by SAP as potentially breaking (release note) |

## Version-sensitive themes (register)

Tracked in detail in [references/sap-cap-sources.md](../references/sap-cap-sources.md) §version-sensitive; headline items:

- **Defaults that flipped:** JSON log formatter production default (cds ≥ 7.5); production deny-by-default auth (`restrict_all_services`); CAP Java 4+: deep authorization & input-data checking on by default; CAP Java 5 default flips (`preferServiceException`, outbox ordering, auth mode, HANA fuzzy search); cds 10 makes former opt-in flags the defaults.
- **Package migrations:** built-in DB services → `@cap-js/{sqlite,hana,postgres}` (cds 8+); `cds.test` → `@cap-js/cds-test` (cds 8+); MTX classic → `@sap/cds-mtxs` (mandatory since CAP Java 3); `cds-feature-xsuaa` → `cds-feature-identity`; Jest support being abandoned → Vitest/node test runner; ESM default for new Node projects (cds 10).
- **Deprecated/removed formats:** `hdbcds` deploy format (unusable ≥ cds 9); `install-cdsdk` Maven goal (removed in cds-maven-plugin 5).
- **Identity direction:** SAP now says "start new projects with IAS" (+ AMS); XSUAA positioned as legacy with cross-consumption (Jan 2026 authentication guide).
- **Migration tooling:** `cds upgrade` (Node, **alpha** — manual review required); OpenRewrite recipes `com.sap.cds:cds-services-recipes` (Java), documented migration chain Java 1.x→…→5.0. **Upgrade hazard:** cds 8→10 directly risks double-processing of queued/outbox messages — drain the queue or pass through v9 (SAP event-queues guide).
- **Messaging & MTX direction:** SAP Cloud Application Event Hub is "the new default offering" (Event Mesh legacy); old MTX (`@sap/cds-mtx`) maintenance-only ≤ CAP Java 2.x — streamlined `@sap/cds-mtxs` mandatory; `cds.outboxed` retained only as synonym of `cds.queued`.
- **AI/MCP:** MCP protocol adapter **beta** (`@mcp` annotation, `@cap-js/mcp` / `cds-adapter-mcp`); `@cap-js/mcp-server` dev-time assistant (no stability label); AI Core plugin/agents/HCQL alpha-beta — all unstable, re-verify before standardizing.

## Upgrade & migration approach (ORG)

1. Watch monthly release notes; assess impact against the version-sensitive register.
2. For major upgrades: read the official upgrade/migration guide for the runtime, run official tooling (`cds upgrade` / OpenRewrite recipes), then manually review — SAP itself flags the Node tool as alpha with false positives.
3. Update *every* version pin location together: `package.json` engines / `.nvmrc` / `pom.xml`, Dockerfiles, CI matrices, BTP runtime config (per SAP upgrade guide).
4. Re-run the full test suite and a milestone-M8-scoped review after any major upgrade.
5. After EOL of a CAP major without upgrade: freeze dependencies (SAP policy) and treat the situation as a Critical `CAP-VER` finding until resolved.
