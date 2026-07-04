# Test Strategy — Restful Booker API Automation

**Author:** Qasim Mahmood, Senior SDET
**System under test:** Restful Booker (https://restful-booker.herokuapp.com) — a public hotel-booking practice API
**Tooling:** Postman (Collection v2.1) + Newman ^6 + newman-reporter-htmlextra + GitHub Actions

---

## 1. Scope & Objectives

**In scope**

- Functional API testing of the booking service: health check, token-based authentication, and the full booking CRUD lifecycle (`/ping`, `/auth`, `/booking`).
- Contract validation of response payloads via JSON Schema.
- Negative testing of input validation and authorization enforcement.
- Basic non-functional guardrail: per-request response-time SLA.

**Out of scope**

- UI testing (the API has no first-party UI), load/stress testing, security penetration testing, and testing of Heroku platform behavior (cold starts are handled via SLA tuning, not asserted against).

**Objectives**

1. Detect functional regressions in every CRUD operation and the auth flow.
2. Detect contract drift — any change to response structure or field types fails the run.
3. Verify the API rejects invalid input and unauthorized mutations.
4. Provide fast, trustworthy signal in CI: deterministic runs, machine-readable results (JUnit), human-readable evidence (htmlextra HTML).

## 2. Test Pyramid Positioning

This suite sits at the **API/service layer** of the test pyramid — above unit tests (owned by the API's developers, not reproducible from the outside) and below E2E/UI tests (not applicable here). API-layer tests are the sweet spot for a black-box consumer: fast enough to run on every push, stable enough for a nightly cron against a live third-party service, and precise enough to pin down *which* endpoint and *which* contract field broke. The suite is intentionally a regression pack, not an exploratory tool: every test is deterministic and self-contained per run.

## 3. Test Design Techniques

| Technique | Where applied |
|---|---|
| **State-transition testing** | The Booking CRUD folder models the resource lifecycle as an explicit chain: create → read → update (PUT) → partial update (PATCH) → delete → verify-404. State (booking ID, auth token) flows between requests via collection variables, so the chain proves each transition — including the terminal state — rather than testing endpoints in isolation. |
| **Equivalence partitioning & boundary values** | `data/booking-test-data.json` defines 3 booking profiles spanning partitions: a typical booking, an edge-value profile (zero/large price, minimal-length names, boundary dates, empty `additionalneeds`), and a distinct valid variant. Each iteration exercises the full lifecycle. |
| **Negative testing** | Dedicated folder: invalid field types, missing required fields, syntactically malformed JSON, wrong credentials, and mutation attempts with invalid tokens. Authorization negatives (403 on PUT/DELETE without a valid token) are treated as the highest-value cases. |
| **Schema/contract validation** | Every key response is asserted with `pm.response.to.have.jsonSchema(...)` (ajv under the hood): required fields, types, and nested `bookingdates` structure. This catches silent contract drift that status-code checks miss. |
| **Performance thresholds** | A collection-level test asserts `pm.response.responseTime` against the `maxResponseTimeMs` environment variable. Defining it once at collection level and parameterizing per environment avoids scattering magic numbers and lets the local Docker environment use a tighter budget than the shared Heroku instance. |
| **Documenting known defects** | Restful Booker intentionally deviates from HTTP convention (`/ping` → 201, DELETE → 201, 500 where 400 is correct). Tests assert the actual behavior with comments naming the deviation. Rationale: a test that asserts the "correct" code would be permanently red and train people to ignore failures; asserting reality means a future fix surfaces as a conscious, reviewed assertion update. |

## 4. Test Data Management

- **Generated, not hardcoded.** Pre-request scripts build each booking payload at runtime and store it in a collection variable; all assertions compare the response against that stored payload. No expected value is duplicated as a string literal in a test script, so data changes never require assertion changes.
- **Dual-source with fallback.** Scripts read from Newman iteration data (`pm.iterationData`) when a data file is supplied (`-d`), and fall back to Postman dynamic variables (`$randomFirstName`, etc.) otherwise. One collection therefore serves three modes: interactive runs in the Postman app, single CI smoke/regression runs, and data-driven regression (`npm run test:data`, 3 iterations).
- **Self-cleaning.** The lifecycle deletes what it creates and verifies the deletion, so repeated runs do not accumulate state on the shared instance.
- **No secrets.** Restful Booker's admin credentials are public documentation values; they live in the environment files by design, which keeps the repo fork-and-run.

## 5. Environments Strategy

| Environment | Target | Purpose |
|---|---|---|
| `production` | `restful-booker.herokuapp.com` | Default CI target; nightly cron detects upstream drift |
| `local` | `localhost:3001` (Docker: `mwinteringham/restfulbooker`) | Stable, isolated target for development and debugging; immune to shared-instance noise and Heroku cold starts |

Environments differ **only in data** (`baseUrl`, `maxResponseTimeMs`, credentials) — never in test logic. Anything environment-specific must be expressed as an environment variable, which is what makes the same collection portable across targets and keeps CI's `workflow_dispatch` environment-choice input a one-flag switch.

## 6. Reporting

- **CLI reporter** for immediate console feedback locally and readable CI logs.
- **htmlextra** for human-readable evidence: per-request assertions, request/response bodies, and per-iteration breakdowns — the artifact you attach to a bug report.
- **JUnit XML** for machine consumption: CI test annotations today, dashboards/test-management ingestion tomorrow.
- In CI, both reports are uploaded as artifacts (`newman-html-report`, `newman-junit-results`, 30-day retention) and a summary table is written to the GitHub Actions step summary so results are visible without downloading anything.

## 7. CI Integration

The `API Tests` workflow (`.github/workflows/api-tests.yml`):

- **Triggers:** push and PR to `main` (regression gate), nightly cron at 02:30 UTC (drift detection against the live API), and `workflow_dispatch` with an environment input (on-demand runs against either target).
- **Steps:** checkout → Node 22 setup → `npm ci` (lockfile-pinned toolchain) → full collection run with CLI + htmlextra + JUnit reporters → a separate data-driven step running 3 iterations from the data file → artifact upload (also on failure — failed runs are when reports matter most) → step-summary results table.
- **Fork-friendly by design:** no repository secrets are required, so a fork's first push gets a green (or honestly red) pipeline with zero setup.

## 8. Known Limitations & Future Improvements

**Limitations**

- The nightly target is a shared public instance: transient 5xx/latency noise from other users or Heroku cold starts can cause flakes unrelated to code changes.
- The CRUD chain is sequential by design; a mid-chain failure cascades into downstream failures (mitigated by the HTML report making the first failure obvious).
- Schema definitions are maintained by hand inside the collection rather than derived from a source-of-truth spec.
- SLA assertions are a guardrail, not a performance test — single-sample response times against a shared host are indicative only.

**Planned improvements**

1. **Spec-driven contract tests** — generate schemas/tests from an OpenAPI definition so the contract has a single source of truth.
2. **Parallel execution** — run independent folders (or the data-driven iterations) concurrently via `newman.run` in a Node script or a CI matrix, cutting wall-clock time as the suite grows.
3. **Mock-based isolation** — a Prism/WireMock mock layer for negative and contract cases, removing dependence on the live instance for the majority of the suite.
4. **Quality gates** — fail the pipeline on assertion-pass-rate or p95 response-time thresholds parsed from the JSON reporter output, rather than binary pass/fail alone.
5. **Flake management** — automatic retry-with-report for known-transient failures (Heroku cold start on the first `/ping`), keeping the signal honest without masking real regressions.
