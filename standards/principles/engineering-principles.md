# Layer 1 — Engineering Principles

Ten principles that define how we build CAP applications. They are not generic software slogans: each is grounded in official CAP documentation where possible (authority per [docs/authority-levels.md](../../docs/authority-levels.md), sources in [references/sap-cap-sources.md](../../references/sap-cap-sources.md)) and is operationalized by named Layer 2 rule categories. Principles resolve rule conflicts and govern situations no rule covers.

> URLs verified 2026-08-11. CAP's documentation was restructured in January 2026; older `/docs/about/*` URLs have moved.

---

## P1 — Domain-first modeling
**Capture intent, not implementations.** The domain model is the center of a CAP application: CDS models capture *what*, not *how*, and fuel CAP's generic runtimes. Domain logic lives in the model wherever the model can express it (types, associations, compositions, annotations) before any code is written.
*Grounding:* SAP-REC — "we want to capture intent, not imperative implementations — that is: What, not How" ([Domain Modeling](https://cap.cloud.sap/docs/guides/domain/)); CDS as the universal modeling language ([Core Concepts](https://cap.cloud.sap/docs/get-started/concepts)).
*Operationalized by:* `CAP-CDS`, `CAP-ARCH`.

## P2 — CAP-native first
**Use the golden path before building beside it.** CAP serves recurring requirements — CRUD, drafts, validation, authorization, pagination, i18n, media, messaging, multitenancy — "out of the box by generic providers". Whenever CAP provides an established capability, we use it; custom implementations of solved problems require documented justification. Corollary from SAP's own anti-patterns: **never wrap CAP in your own abstraction layer** ("abstracting from an abstraction is a bad idea"), and don't rebuild what CAP intentionally omits (element-level determination frameworks, Active Records, code generation).
*Grounding:* SAP-REC — [Introduction to CAP](https://cap.cloud.sap/docs/get-started/features); [Bad Practices](https://cap.cloud.sap/docs/about/bad-practices) (page relocating after Jan 2026 restructure).
*Operationalized by:* all `CAP-*` categories; enforced on AI work by `AI-DEV-004/-011/-012`.

## P3 — Services as use-case facades
**Every active thing is a service — and each service serves one use case.** Services are stateless, process passive data, and expose denormalized projections of the domain model tailored to a specific use case and user group. SAP explicitly flags "one service exposing all entities 1:1" as an anti-pattern.
*Grounding:* SAP-REC ("strongly recommend") — [Providing Services](https://cap.cloud.sap/docs/guides/services/providing-services); service principles in [Core Concepts](https://cap.cloud.sap/docs/get-started/concepts).
*Operationalized by:* `CAP-SRV`, `CAP-ARCH`, `CAP-SEC` (exposure control).

## P4 — Separation of concerns, CAP-style
**Aspects keep the core model clean.** Authorization, UI annotations, localization, and other secondary concerns live in separate files/aspects with their own ownership and lifecycle; consumers see effective models. Business logic sits in event handlers on services — not sprinkled through models, not hidden in helper frameworks.
*Grounding:* SAP-REC — [Aspect-Oriented Modeling](https://cap.cloud.sap/docs/cds/aspects); [Domain Modeling](https://cap.cloud.sap/docs/guides/domain/).
*Operationalized by:* `CAP-CDS`, `CAP-LOGIC`.

## P5 — Agnostic core, late decisions
**Stay platform-, protocol-, and topology-agnostic.** Application code consumes databases, messaging, auth, and remote services through CAP's uniform service abstractions, preserving fast local development (SQLite/H2, mocks, "airplane mode"), environment parity, and "late-cut microservices": start modulithic, cut deployment units when the architecture is clear — by configuration, not rewrite.
*Grounding:* SAP-REC — agnostic design & late-cut microservices ([Introduction](https://cap.cloud.sap/docs/get-started/features), [Core Concepts](https://cap.cloud.sap/docs/get-started/concepts), [Microservices](https://cap.cloud.sap/docs/guides/deploy/microservices)).
*Operationalized by:* `CAP-ARCH`, `CAP-DB`, `CAP-INT`, `CAP-DEP`.

## P6 — Secure by design, deny by default
**Security is modeled, not bolted on.** Authorization is declared in the model (`@requires`/`@restrict`, instance-based restrictions) and designed early; production always runs real identity services (SAP: start new projects with IAS); mock/dummy auth never reaches production; every exposure — entity, association, composition, action — is a deliberate decision. CAP's secure defaults (authenticated endpoints, sanitized errors, masked logs) are kept, not disabled.
*Grounding:* SAP-REQ/SAP-REC — [Authorization](https://cap.cloud.sap/docs/guides/security/authorization), [Authentication](https://cap.cloud.sap/docs/guides/security/authentication), [Data Protection](https://cap.cloud.sap/docs/guides/security/data-protection).
*Operationalized by:* `CAP-SEC`, `CAP-PRIV`, `CAP-MT`.

## P7 — Testable business behavior
**Behavior is proven through service interfaces.** What the application promises is tested where consumers see it — through services (via `cds.test` / CAP Java's layered test support) — including denial and rejection paths, on fast in-memory infrastructure. Untested behavior is unfinished behavior.
*Grounding:* SAP-REC — [cds.test best practices](https://cap.cloud.sap/docs/node.js/cds-test), [CAP Java testing](https://cap.cloud.sap/docs/java/developing-applications/testing).
*Operationalized by:* `CAP-TEST`; enforced on AI work by `AI-TEST-*`.

## P8 — Fail loudly, operate visibly
**Errors are honest; operations are observable.** Follow SAP's "let it crash": don't catch what you can't handle, never hide errors and continue. Client errors use CAP's error APIs (protocol-correct, localized, sanitized in production). Production runs structured JSON logs with correlation IDs, telemetry, and health endpoints — a request must be traceable end to end.
*Grounding:* SAP-REC/SAP-REQ — [Node.js best practices](https://cap.cloud.sap/docs/node.js/best-practices), [Indicating errors (Java)](https://cap.cloud.sap/docs/java/event-handlers/indicating-errors), [cds.log](https://cap.cloud.sap/docs/node.js/cds-log), [Observability (Java)](https://cap.cloud.sap/docs/java/operating-applications/observability).
*Operationalized by:* `CAP-ERR`, `CAP-LOG`, `CAP-OPS`.

## P9 — Performance through the model
**Performance problems are prevented in the model and the query, not patched in code.** Let the database work: push queries down (CQL, projections, `$expand` over static JOIN views), keep pagination limits on, keep calculated fields out of filter/sort paths, stream media, avoid element-level processing frameworks — the pattern SAP built CAP to escape.
*Grounding:* SAP-REC — [Performance considerations for CDS modeling](https://cap.cloud.sap/docs/guides/databases/performance), [served out of the box](https://cap.cloud.sap/docs/guides/services/served-ootb).
*Operationalized by:* `CAP-PERF`, `CAP-DB`, `CAP-CDS`.

## P10 — Production-ready and evolvable
**Every release is deployable, supported, and upgradeable.** Projects stay on Active CAP majors (SAP maintenance model: Maintenance ≤ 12 months, then EOL), with frozen-but-refreshed dependencies, reproducible pipeline deployments, health checks wired, and version pins consistent everywhere. Maintainability includes upgradability: the standard's [version policy](../../docs/version-management.md) is part of production readiness.
*Grounding:* SAP-REQ/SAP-REC — [Release schedule](https://cap.cloud.sap/docs/releases/schedule), [Deploy to CF](https://cap.cloud.sap/docs/guides/deploy/to-cf), [CAP Java versions](https://cap.cloud.sap/docs/java/versions).
*Operationalized by:* `CAP-VER`, `CAP-DEP`, `CAP-CICD`, `CAP-OPS`.

---

## Using the principles

- **Rule conflict:** the principle closer to the domain wins (P1–P4 over P9–P10) unless security (P6) is involved — security always wins.
- **No applicable rule:** evaluate the situation against P1–P10 and record the reasoning (`AI-REC` label if AI-generated) per [AI-DOC-001](../ai/ai-documentation-rules.md).
- **Proposing new rules:** every new Layer 2 rule must name the principle(s) it operationalizes.
