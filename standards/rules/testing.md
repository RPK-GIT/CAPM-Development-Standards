# CAP-TEST — Testing

Part of the [Layer 2 rule catalog](README.md). Rules follow the [rule template](../../templates/rule-template.md) and the catalog invariants. Candidate dispositions: [references/candidate-dispositions.md](../../references/candidate-dispositions.md). Related ORG gap: G-01 (coverage thresholds, pyramid ratios — SAP prescribes none; none are invented here).

**Rules:** 7 active (0 Critical, 1 High, 5 Medium, 1 Low). All SAP references verified against official CAP documentation on **2026-08-12**.

Testing rules state **what must be verified and how test infrastructure is built** — application behavior itself is governed by the implementation rules they cross-reference. The Layer 3 [AI-TEST rules](../ai/ai-testing-rules.md) govern how Claude Code writes tests; this category governs what the *project's suite* must look like regardless of author.

| ID | Title | Severity | Authority | Runtime |
|---|---|---|---|---|
| CAP-TEST-001 | Bootstrap and isolate Node.js tests with `cds.test` | Medium | SAP-REQ (bootstrap order) / SAP-REC (isolation) | Node.js |
| CAP-TEST-002 | Test CAP Java along the documented layers | Medium | SAP-REC | Java |
| CAP-TEST-003 | Assert stable contracts, not incidental details | Medium | SAP-REC | Node.js |
| CAP-TEST-004 | Keep tests portable across test runners | Low | SAP-REC | Node.js |
| CAP-TEST-005 | Test authenticated flows with mock users | Medium | SAP-REC | Both |
| CAP-TEST-006 | Cover cloud-only behavior with hybrid tests | Medium | SAP-REC | Both |
| CAP-TEST-007 | Verify security behavior by test | High | SAP-REC | Both |

---

## CAP-TEST-001 — Bootstrap and isolate Node.js tests with `cds.test`

| Field | Value |
|---|---|
| **Rule ID** | CAP-TEST-001 |
| **Title** | Bootstrap and isolate Node.js tests with `cds.test` |
| **Category** | Testing |
| **Severity** | Medium |
| **Authority** | SAP-REQ (bootstrap order: "always ensure your calls to `cds.test(...)` go first") / SAP-REC (isolation pattern) |
| **Applicability** | Node.js projects with automated tests |
| **Runtime** | Node.js (Java counterpart: CAP-TEST-002) |
| **CAP version** | ⏱ `@cap-js/cds-test` is the current test package (extracted from `@sap/cds` with cds 8) — added as devDependency |
| **Status** | Active |
| **Related rules** | CAP-TEST-003, CAP-TEST-004, CAP-DB-002 (in-memory dev DB), AI-TEST-002 |
| **Last verified** | 2026-08-12 |

### Rule statement
Node.js test suites MUST use `cds.test` from `@cap-js/cds-test` (devDependency), and the `cds.test()`/`cds.test.in(folder)` call MUST be the first thing the test executes — SAP: "always ensure your calls to `cds.test.in(folder)` or `cds.test(folder)` goes first, before anything else loading `cds.env`". Tests SHOULD run against the in-memory database and reset state per test via `data.reset()` in a `beforeEach` hook so every test starts from known data.

### Rationale
`cds.test()` first is what makes `cds.env` (and thereby all plugins, profiles, and database wiring) load from the test's target folder — code touching `cds.env` before it silently binds the wrong configuration, producing tests that pass against the wrong setup or fail inexplicably. Per-test `data.reset()` removes inter-test coupling, the primary source of order-dependent, flaky suites. **Medium:** broken test infrastructure undermines the suite's evidence value but doesn't touch production.

### Evidence expected in code
`@cap-js/cds-test` in devDependencies; test files starting with the `cds.test(...)` call before any `cds.*` access; `data.reset()` (or equivalent deliberate strategy) in shared hooks.

### Detection guidance
1. Check `package.json` devDependencies for `@cap-js/cds-test` → absent while CAP tests exist → FAIL.
2. For each test file using `cds.test`: verify the call precedes any other `cds` usage/imports that load `cds.env` → violation → FAIL with file:line.
3. Check state isolation: `data.reset()` in `beforeEach` or a documented alternative (per-file deploys, isolated fixtures) → shared mutable data across tests with no strategy → FAIL.
4. Run the suite in a shuffled/isolated order if cheaply possible; order-dependent failures confirm findings (else static evidence suffices).
5. NOT APPLICABLE for projects without Node.js tests (their absence is a lifecycle-gate concern, not this rule).

### Good example
```js
const cds = require('@sap/cds');
const { GET, expect, data } = cds.test(__dirname + '/..');  // FIRST — before anything touching cds.env
data.reset();                                                // fresh data per test (beforeEach)

it('serves books', async () => {
  const { data: result } = await GET('/odata/v4/catalog/Books');
  expect(result.value).to.containSubset([{ ID: 201 }]);
});
```

### Bad example
```js
const cds = require('@sap/cds');
const model = cds.env.requires.db;      // touches cds.env BEFORE cds.test —
const test = cds.test(__dirname);        // env now loaded from the wrong folder
let created;                             // shared state mutated across tests:
it('creates', async () => { created = await POST(…); });
it('reads back', () => GET(`/Books(${created.ID})`));   // order-dependent
```

### Exception guidance
Suites not using `cds.test` at all (pure unit tests of plain functions) are out of scope. A deliberate cross-test fixture (expensive setup) is acceptable when read-only for all tests and documented in the suite.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-test (`@cap-js/cds-test` devDependency; bootstrap-first requirement; `data.reset()` in `beforeEach`)

---

## CAP-TEST-002 — Test CAP Java along the documented layers

| Field | Value |
|---|---|
| **Rule ID** | CAP-TEST-002 |
| **Title** | Test CAP Java along the documented layers |
| **Category** | Testing |
| **Severity** | Medium |
| **Authority** | SAP-REC (layer guidance); SAP-REQ elements: H2 supported only up to 2.3.x; MTX scenarios not testable on H2 |
| **Applicability** | CAP Java projects with automated tests |
| **Runtime** | Java (Node.js counterpart: CAP-TEST-001) |
| **CAP version** | ⏱ H2 "support until version 2.3.x" per current docs; RestTestClient requires a current Spring stack — re-verify at majors |
| **Status** | Active |
| **Related rules** | CAP-TEST-005, CAP-TEST-006 (MTX needs hybrid tests), CAP-MT rules, AI-TEST-002 |
| **Last verified** | 2026-08-12 |

### Rule statement
CAP Java suites SHOULD be structured along the documented layers: (1) unit tests on handler logic with mocked dependencies (SAP: "a solid first layer of your testing efforts"); (2) service-layer tests running CQN statements/events against the service layer directly; (3) integration tests with `@SpringBootTest` plus MockMvc or RestTestClient — documented as equivalent server-side roundtrips "through the whole CAP architecture" ("there's no preference"). Constraints MUST be respected: H2 is supported only up to version 2.3.x, and multitenancy/extensibility (MTXS) scenarios are **not testable on H2** — they require hybrid testing against cloud services (CAP-TEST-006).

### Rationale
The layered approach is SAP's documented calibration of speed vs coverage: pure-logic defects caught in milliseconds, wiring and adapter behavior in the integration roundtrip. The two constraints are hard facts, not style: a newer H2 or an H2-based MT test produces false confidence in unsupported territory. **Medium:** suite architecture quality; the constraints, violated, invalidate exactly the affected tests.

### Evidence expected in code
JUnit tests at multiple layers (`srv/src/test/java/**`: plain unit tests; `@SpringBootTest` classes with `@AutoConfigureMockMvc` or RestTestClient); H2 pinned ≤ 2.3.x in `pom.xml`; no MT/extensibility assertions running on H2.

### Detection guidance
1. Inventory test classes: unit-level (no Spring context) and integration-level (`@SpringBootTest`) both present → PASS structure; only one layer for a non-trivial app → observation/FAIL (Medium) depending on scope of untested logic.
2. Check `pom.xml` H2 version: > 2.3.x → FAIL (documented support boundary).
3. Search test sources for multitenancy/extensibility scenarios (tenant subscription, extension activation) executed against H2 → FAIL (documented impossibility) — hybrid coverage required instead (CAP-TEST-006).
4. Verify integration tests actually roundtrip the CAP stack (MockMvc/RestTestClient against OData endpoints), not just Spring context loading.
5. NOT APPLICABLE without Java tests (lifecycle-gate concern otherwise).

### Good example
```java
// layer 1: pure handler logic, mocked persistence
@Test void discountAboveThreshold() { assertEquals("11%", new Pricing().discount(101)); }

// layer 3: full CAP roundtrip
@SpringBootTest @AutoConfigureMockMvc
class CatalogIT {
  @Autowired MockMvc mvc;
  @Test void serveBooks() throws Exception {
    mvc.perform(get("/api/browse/Books")).andExpect(status().isOk());
  }
}
```

### Bad example
```xml
<!-- pom.xml — H2 beyond the documented support boundary -->
<dependency><groupId>com.h2database</groupId><artifactId>h2</artifactId>
  <version>2.4.240</version></dependency>
```

### Exception guidance
Small services may legitimately skip the middle (service-layer) tier — SAP presents layers as an approach, not a quota. The H2 boundary and the MTX limitation have no exceptions; the alternative is hybrid testing.

### SAP reference
- https://cap.cloud.sap/docs/java/developing-applications/testing (layers; MockMvc/RestTestClient equivalence; "we only support until version 2.3.x of H2"; MTXS not testable on H2 → hybrid testing)

---

## CAP-TEST-003 — Assert stable contracts, not incidental details

| Field | Value |
|---|---|
| **Rule ID** | CAP-TEST-003 |
| **Title** | Assert stable contracts, not incidental details |
| **Category** | Testing |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | All Node.js CAP tests (the principle applies to Java suites as general practice — SAP documents it on the Node testing page) |
| **Runtime** | Node.js |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-TEST-001, CAP-ERR-005 (message texts are localized — another reason not to assert them), AI-TEST-001 |
| **Last verified** | 2026-08-12 |

### Rule statement
Test assertions SHOULD check only what the test is about — SAP: "only check what's really relevant for the functionality you're testing. Make minimal assumptions about irrelevant details." Concretely: assert **stable error codes**, never error-message texts ("Check for guaranteed stable error codes instead"); avoid full-payload/snapshot assertions in favor of essential-subset checks (SAP's named alternative: `containSubset`); never assert on timestamps, generated IDs (beyond existence), or ordering the contract doesn't promise.

### Rationale
Over-specified assertions make the suite a change-detector instead of a behavior-verifier: localized or reworded messages (CAP-ERR-005), framework field additions, and formatting changes break tests that verified nothing about the application's promises — and teams respond by weakening real checks. **Medium:** brittleness silently erodes the suite's evidence value, which is the suite's entire purpose.

### Evidence expected in code
Assertions on status codes, stable error `code` values, and essential payload subsets; absence of message-text equality checks and full-response snapshots.

### Detection guidance
1. Search test files for message-text assertions: equality/matching against human-readable error strings (`expect(...message).to.equal('The order …')`) → FAIL per site (assert the `code`).
2. Search for snapshot assertions (`toMatchSnapshot`, stored full-payload fixtures compared wholesale) on CAP responses → FAIL (subset alternative).
3. Search for assertions on volatile fields (timestamps equality, `modifiedAt`, generated UUID values) → FAIL per site.
4. Confirm the positive pattern is available (`containSubset` via `cds.test.expect` or equivalent) and used for structured checks.
5. Report with file:line; systematic occurrences → category FAIL, isolated ones → findings.

### Good example
```js
const { status, data } = await POST('/odata/v4/admin/Books', badPayload, { auth });
expect(status).to.equal(400);
expect(data.error.code).to.equal('ASSERT_RANGE');       // stable code, not text
expect(data.value ?? data).to.containSubset({ title: 'Wuthering Heights' });
```

### Bad example
```js
expect(data.error.message).to.equal('Value 100000 is not in specified range [0, 99999]');
// breaks on rewording and on every non-default Accept-Language
expect(response).toMatchSnapshot();   // change detector: any new field fails the suite
```

### Exception guidance
Contract tests whose explicit purpose is the full wire format (e.g., a frozen external API contract) legitimately assert complete payloads — mark the test as a contract test. Asserting a message text is acceptable only where the text itself is the tested feature (i18n bundle tests).

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-test (minimal assertions; stable error codes; `containSubset` over snapshots)

---

## CAP-TEST-004 — Keep tests portable across test runners

| Field | Value |
|---|---|
| **Rule ID** | CAP-TEST-004 |
| **Title** | Keep tests portable across test runners |
| **Category** | Testing |
| **Severity** | Low |
| **Authority** | SAP-REC ("it's recommended to avoid using runner-specific features and stick to the common APIs provided by `cds.test`") |
| **Applicability** | Node.js CAP test suites |
| **Runtime** | Node.js |
| **CAP version** | Documented runners per current docs: Node's built-in test runner, Vitest, Jest, Mocha (no deprecation of any runner is documented — the page notes Jest friction with ESM/current Chai only) |
| **Status** | Active |
| **Related rules** | CAP-TEST-001, CAP-TEST-003 |
| **Last verified** | 2026-08-12 |

### Rule statement
Node.js CAP tests SHOULD restrict themselves to the common runner APIs (`describe`, `it`/`test`, `before/afterAll`, `before/afterEach`) and the assertion utilities provided by `cds.test` (`cds.test.expect`), avoiding runner-specific features — so the project can move between the documented runners (Node test runner, Vitest, Jest, Mocha) without rewriting tests.

### Rationale
SAP explicitly recommends runner portability and documents concrete friction (Jest with ESM and current Chai versions) — projects locked into runner-specific APIs pay a rewrite when the ecosystem shifts under them. **Low:** pure maintainability; nothing functional depends on it.

### Evidence expected in code
Tests using common APIs and `cds.test.expect`; no reliance on runner-exclusive features (runner-specific mock frameworks, snapshot systems, custom matchers) without recorded intent.

### Detection guidance
1. Identify the runner(s) in use (package.json scripts, config files).
2. Search tests for runner-exclusive APIs (e.g., `jest.mock`, `vi.mock`, `jest.fn` used pervasively, runner-specific snapshot/matcher extensions) → each an observation; pervasive use → FAIL (Low).
3. Check assertions use `cds.test.expect`/standard Chai-style APIs.
4. Report representative sites.

### Good example
```js
const { expect, GET } = cds.test(__dirname);   // portable across documented runners
describe('catalog', () => {
  it('serves books', async () => expect((await GET('/odata/v4/catalog/Books')).status).to.equal(200));
});
```

### Bad example
```js
jest.mock('../srv/lib/pricing');                      // runner-locked mocking
expect(books).toMatchInlineSnapshot(`…`);             // runner-specific snapshots
```

### Exception guidance
A deliberate, recorded commitment to one runner (team standard) relaxes this rule to an observation — the trade-off is theirs to own. Runner-specific mocking may be genuinely needed for module-level isolation; keep it localized.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-test (portability recommendation; documented runners; Jest ESM/Chai friction)

---

## CAP-TEST-005 — Test authenticated flows with mock users

| Field | Value |
|---|---|
| **Rule ID** | CAP-TEST-005 |
| **Title** | Test authenticated flows with mock users |
| **Category** | Testing |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | Projects with restricted services (i.e., nearly all — cross-ref CAP-SEC-001) |
| **Runtime** | Both (mechanics differ: Node.js `cds.test` auth defaults with mock users like `alice`; Java: Spring Security's `@WithMockUser` / CAP mock-user configuration — no CAP-specific `@MockUser` annotation is documented) |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-SEC-002 (mock auth stays out of production), CAP-TEST-007, AI-TEST-004 |
| **Last verified** | 2026-08-12 |

### Rule statement
Tests of authenticated/authorized behavior MUST use the framework's mock-user mechanisms — Node.js: request auth defaults with mocked users (e.g., `defaults.auth = { username: 'alice', … }`); Java: Spring Security test support (`@WithMockUser`) and CAP mock-user configuration in test profiles — never real credentials, tokens, or a live IdP in the default suite. Mock users MUST be modeled with the roles/attributes the scenario needs, so authorization logic (not just authentication plumbing) is exercised.

### Rationale
Mock users make security behavior testable in the inner loop: deterministic identities with exactly the roles under test, no credential material in the repository (CAP-SEC-017), no IdP dependency in CI. Suites that skip authenticated flows because "auth is hard to test" leave the authorization model unverified — the gap CAP-TEST-007 measures. **Medium:** test-infrastructure enablement; the verification duty itself is CAP-TEST-007 (High).

### Evidence expected in code
Test-profile mock users with scenario roles; tests exercising restricted operations as different users; no real tokens/secrets in test code or fixtures.

### Detection guidance
1. Check test configuration for mock users with roles (Node: `cds.requires.auth.users` in test profile / `cds.test` auth defaults; Java: `@WithMockUser(roles=…)` usage or `cds.security.mock.users` in test config).
2. Verify restricted services are tested with authenticated requests (auth options present in test calls) → suites calling only unauthenticated/public endpoints while restricted services exist → FAIL (feeds CAP-TEST-007).
3. Search test code/fixtures for real-looking credentials/tokens → FAIL here and report under CAP-SEC-017.
4. Verify mock users carry the roles/attributes the scenarios assert (not one super-user for everything — that never exercises denial; cross-ref CAP-TEST-007).
5. NOT APPLICABLE only for projects with no restricted services (rare; verify against CAP-SEC-001 results).

### Good example
```js
// Node.js — distinct users per role scenario
const admin = { auth: { username: 'alice' } };          // admin role in test config
const viewer = { auth: { username: 'bob' } };           // viewer role
await POST('/odata/v4/admin/Books', book, admin);        // allowed
const { status } = await POST('/odata/v4/admin/Books', book, viewer);
expect(status).to.equal(403);                            // denied
```

### Bad example
```js
// real service key in the test repo + a single almighty test user:
// denial paths never exercised, credentials leaked
const token = 'eyJhbGciOiJSUzI1NiIsInR5cCI6...';
await GET('/odata/v4/admin/Books', { headers: { Authorization: `Bearer ${token}` } });
```

### Exception guidance
Dedicated end-to-end/staging suites may use real (test-tenant) identities via bound credentials at runtime (`cds bind --exec`, CI secrets) — never committed, and in addition to mock-user suites, not instead of them.

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-test (auth defaults with mocked users)
- https://cap.cloud.sap/docs/java/developing-applications/testing (MockMvc with mocked users — Spring Security test support)
- https://cap.cloud.sap/docs/node.js/authentication (mocked strategy with pre-defined users)

---

## CAP-TEST-006 — Cover cloud-only behavior with hybrid tests

| Field | Value |
|---|---|
| **Rule ID** | CAP-TEST-006 |
| **Title** | Cover cloud-only behavior with hybrid tests |
| **Category** | Testing |
| **Severity** | Medium |
| **Authority** | SAP-REC |
| **Applicability** | Projects whose behavior depends on features the local stack cannot exercise (HANA-specific features, MTX lifecycle, pessimistic locking, real brokers); NOT APPLICABLE where local parity fully covers the app |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | CAP-DB-002 (parity limits), CAP-DB-008 (locking not on SQLite), CAP-TEST-002 (MTX not on H2), CAP-MT-003 (isolation tests), CAP-SEC-017 (`cds bind` pointer-only credentials) |
| **Last verified** | 2026-08-12 |

### Rule statement
Behavior that local development databases and mocks cannot exercise — documented cases: MTX/multitenancy scenarios (not testable on H2), pessimistic locking (unsupported on SQLite; H2 exclusive-only), HANA-specific features, real-broker delivery — MUST be covered by hybrid tests running against bound cloud services, using the documented mechanism: `cds bind --exec` wrapping the test run (e.g., `cds bind --exec -- node --test`), which resolves credentials at runtime without materializing them (CAP-SEC-017).

### Rationale
The parity rules (CAP-DB-002/-008, CAP-TEST-002) identify where local green tests prove nothing; hybrid testing is SAP's documented answer. Without it, exactly the risky features (locking under concurrency, tenant lifecycle, HANA behavior) ship verified only against a database that doesn't implement them. **Medium:** a coverage-strategy rule — the underlying feature risks are owned by their implementation rules.

### Evidence expected in code
A hybrid test script/CI stage (`cds bind --exec …`) covering the identified parity-sensitive features; an explicit statement (test docs) of which features need hybrid coverage.

### Detection guidance
1. From the implementation-rule reviews, list parity-sensitive features in use (locking calls, MTX, HANA-specific SQL/features, broker delivery assertions).
2. None in use → NOT APPLICABLE (state the checked list).
3. For each, locate hybrid coverage: `cds bind --exec` scripts (package.json, pipeline), or Java hybrid test profiles against bound services → feature with local-only coverage → FAIL per feature.
4. Verify hybrid runs use runtime-resolved bindings (no committed credentials — cross-check CAP-SEC-017).
5. Check the pipeline actually executes the hybrid stage somewhere (scheduled/pre-release if not per-commit) → never executed → FAIL (dead script).

### Good example
```jsonc
// package.json — hybrid stage for HANA/locking behavior
{ "scripts": { "test": "node --test",
    "test:hybrid": "cds bind --exec -- node --test test/hybrid/" } }
```

### Bad example
```text
Queue-worker uses SELECT ... forUpdate with SKIP_LOCKED semantics;
the only tests run on SQLite where locking is unsupported — the
concurrency behavior shipping to production has never executed in CI.
```

### Exception guidance
Cost/access constraints may limit hybrid runs to pre-release cadence — acceptable when recorded and when the covered features list is explicit. No exception for shipping MT lifecycle changes with zero hybrid verification (gate concern at M7/M8).

### SAP reference
- https://cap.cloud.sap/docs/node.js/cds-test (`cds bind --exec -- node --test` for cloud-service tests)
- https://cap.cloud.sap/docs/java/developing-applications/testing (hybrid testing for MTXS/HANA scenarios)
- https://cap.cloud.sap/docs/tools/cds-bind (runtime-resolved bindings)

---

## CAP-TEST-007 — Verify security behavior by test

| Field | Value |
|---|---|
| **Rule ID** | CAP-TEST-007 |
| **Title** | Verify security behavior by test |
| **Category** | Testing |
| **Severity** | High |
| **Authority** | SAP-REC for the authentication element (SAP: "Don't forget to add authentication tests to ensure properly configured security in your deployed application that rejects unauthenticated requests"); the authorization/role-matrix element is GEN, derived from the CAP-SEC rules it verifies |
| **Applicability** | All projects with restricted services; the tenant element only for multitenant projects |
| **Runtime** | Both |
| **CAP version** | All currently supported versions |
| **Status** | Active |
| **Related rules** | Verifies: CAP-SEC-001 (authorization model), CAP-SEC-005/-015 (unauthenticated rejection), CAP-SEC-010 (instance restrictions), CAP-SEC-012 (validation rejection), CAP-MT-003 (tenant isolation — its test element is *owned there*, referenced here); AI-TEST-004; enabled by CAP-TEST-005 |
| **Last verified** | 2026-08-12 |

### Rule statement
The test suite MUST verify the project's security behavior, not merely its functionality: (1) **unauthenticated rejection** — requests without credentials to protected endpoints are rejected (the SAP-documented authentication test, also the deployed check CAP-SEC-005/-015 require); (2) **authorization matrix** — for each restricted service/operation, at least one allowed case *and* one denied case per distinct role boundary (a suite testing only as an admin user verifies nothing about restrictions); (3) **instance-based restrictions** (where modeled, CAP-SEC-010) — a user sees/modifies only their rows; (4) **validation rejection** — invalid input is rejected per CAP-SEC-012's annotations; (5) for multitenant apps, tenant-isolation tests exist per CAP-MT-003 (owned there; this rule checks their presence in the suite inventory).

### Rationale
Security rules are exactly the behavior that "works" identically whether enforced or not — a missing `@requires` serves data just as smoothly as a present one. Only denial-path tests distinguish the two, and SAP explicitly instructs adding authentication tests for deployed rejection. **High justification:** an untested authorization model means CAP-SEC-001/-010's guarantees are unverified in precisely the way production incidents reveal; this is the "security tests missing for critical authorization paths" case the severity model names — while remaining below Critical because it is missing *verification*, not a demonstrated exposure.

### Implementation guidance
- Drive the matrix from the model: every `@requires`/`@restrict` in the reviewed services yields at least one allow + one deny test (CAP-TEST-005 supplies the users).
- Put the unauthenticated smoke check in the pipeline against the deployed backend URL (satisfies CAP-SEC-015's evidence at the same time).
- Keep denial assertions on status/code (401/403), per CAP-TEST-003.

### Evidence expected in code
Tests asserting 401/403 for unauthenticated and under-privileged access per restricted resource; instance-restriction tests where modeled; validation-rejection tests; MT isolation tests (per CAP-MT-003); a deployed unauthenticated check in the pipeline.

### Detection guidance
1. From CAP-SEC-001's review output, list restricted services/operations and their roles; from CAP-SEC-010, instance restrictions; from CAP-SEC-012, validated elements.
2. Map test coverage: for each restricted resource, find the allow test and the deny test (search for 401/403 assertions with differing auth contexts) → restricted resource with no deny test → FAIL per resource.
3. Locate the unauthenticated-rejection test (suite or pipeline smoke, per CAP-SEC-015 detection) → absent → FAIL.
4. Instance restrictions modeled but untested (no cross-user visibility test) → FAIL per restriction.
5. Validation annotations with no rejection test on security-relevant input → FAIL element (cross-ref CAP-SEC-012 step 5).
6. Multitenant: confirm isolation tests exist (CAP-MT-003's element) → report presence/absence here, findings there.
7. NOT ASSESSABLE only where tests exist but require runtime execution to evaluate — name the run needed.

### Good example
```js
describe('AdminService authorization', () => {
  it('rejects unauthenticated', async () =>
    expect((await POST('/odata/v4/admin/Books', book)).status).to.equal(401));
  it('denies non-admins', async () =>
    expect((await POST('/odata/v4/admin/Books', book, viewer)).status).to.equal(403));
  it('allows admins', async () =>
    expect((await POST('/odata/v4/admin/Books', book, admin)).status).to.equal(201));
});
```

### Bad example
```text
All 214 tests authenticate as 'alice' (admin). Every restricted
operation is exercised — and not a single restriction is verified:
remove every @requires from the model and the suite stays green.
```

### Exception guidance
Genuinely public-only applications (all services deliberately `@requires:'any'` per CAP-SEC-001's documented-public path) reduce this rule to the validation-rejection element. No exception for restricted services without denial tests.

### SAP reference
- https://cap.cloud.sap/docs/guides/security/authentication ("Don't forget to add authentication tests… rejects unauthenticated requests")
- https://cap.cloud.sap/docs/node.js/cds-test / https://cap.cloud.sap/docs/java/developing-applications/testing (mock-user mechanics enabling the matrix)
