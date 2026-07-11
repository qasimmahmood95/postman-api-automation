# Postman API Test Automation — Restful Booker

Production-grade API test automation for the [Restful Booker](https://restful-booker.herokuapp.com) hotel-booking API, built with Postman collections and executed headlessly with Newman in CI.

[![API Tests](https://github.com/qasimmahmood95/postman-api-automation/actions/workflows/api-tests.yml/badge.svg)](https://github.com/qasimmahmood95/postman-api-automation/actions/workflows/api-tests.yml)
![Postman](https://img.shields.io/badge/Postman-Collection%20v2.0-FF6C37?logo=postman&logoColor=white)
![Newman](https://img.shields.io/badge/Newman-%5E6-2E2E2E)
![Node](https://img.shields.io/badge/Node-22.x-339933?logo=node.js&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue)

**[View the live test report](https://qasimmahmood95.github.io/postman-api-automation/)** — the latest htmlextra dashboard, republished by every run on `main`.

## Overview

This repository contains an end-to-end API test suite covering health checks, token-based authentication, the full booking CRUD lifecycle, and a dedicated negative-testing pack against the public Restful Booker API — a deliberately quirky practice API by Mark Winteringham. Every key response is validated against a JSON Schema, all functional assertions compare against dynamically generated test data (never hardcoded literals), and a collection-level response-time SLA is enforced via an environment variable.

The suite is designed the way a production regression pack should be: one collection that serves both single smoke runs and data-driven regression, environment parity between the hosted API and a local Docker instance, deterministic state chaining across the CRUD lifecycle, and a fork-friendly CI pipeline that publishes HTML and JUnit reports on every run — no secrets required.

## Highlights

- **Full CRUD state-transition chain** — create → list (contains new ID) → read → update → patch → delete → verify-404, with state (booking ID, auth token) passed between requests via collection variables.
- **JSON Schema validation** on all key responses (`pm.response.to.have.jsonSchema`, backed by ajv) — contract drift fails the build.
- **Response-time SLA** asserted at collection level, driven by the `maxResponseTimeMs` environment variable — tune per environment, not per request.
- **Dynamic test data** — pre-request scripts pull from Newman iteration data (`-d`) with fallback to Postman dynamic variables, so the same collection runs standalone or data-driven with zero edits.
- **Data-driven regression** — 3 booking profiles (including boundary/edge values) in `data/booking-test-data.json`.
- **Negative-scenario pack** — invalid types, missing required fields, malformed JSON, bad credentials, invalid tokens.
- **Known API quirks documented in tests** — where Restful Booker deviates from HTTP convention, tests assert the *actual* behavior with explanatory comments rather than papering over it.
- **CI/CD with GitHub Actions** — push/PR, nightly cron, manual dispatch with environment choice; HTML + JUnit artifacts and a step-summary results table.

## Test Coverage

| Request | Method | Validations |
|---|---|---|
| **Health Check** | | |
| Ping | GET | Reachability, status 201 (documented quirk), SLA |
| **Authentication** | | |
| Create Token | POST | Status 200, schema, token captured for downstream requests, SLA |
| **Booking CRUD** | | |
| Create Booking | POST | Status 200, schema, echoed payload matches generated data, `bookingid` captured |
| Get Booking IDs | GET | Status 200, schema, non-empty list, contains the newly created `bookingid` |
| Get Booking | GET | Status 200, schema, body matches created data |
| Update Booking (full) | PUT | Status 200, schema, all fields reflect update payload |
| Partial Update | PATCH | Status 200, schema, patched fields changed, others intact |
| Delete Booking | DELETE | Status 201 (documented quirk), SLA |
| Verify Deletion | GET | Status 404 — confirms lifecycle completed |
| **Negative Scenarios** | | |
| Create with invalid types | POST | Actual API behavior asserted (500 where 400 would be correct — documented quirk) |
| Create with missing fields | POST | Error status and error body asserted (500 — documented quirk) |
| Malformed JSON body | POST | Error status asserted |
| Auth with bad credentials | POST | No token issued, error reason asserted |
| Update with invalid token | PUT | Status 403 |
| Delete with invalid token | DELETE | Status 403 |

## Project Structure

```
.
├── .github/
│   └── workflows/
│       └── api-tests.yml                          # CI: Newman run, HTML+JUnit artifacts, nightly cron, manual dispatch
├── collections/
│   └── restful-booker.postman_collection.json     # The suite: Health Check, Authentication, Booking CRUD, Negative Scenarios
├── environments/
│   ├── production.postman_environment.json        # Hosted API: baseUrl, auth creds, maxResponseTimeMs
│   └── local.postman_environment.json             # Restful Booker in Docker (localhost:3001)
├── data/
│   └── booking-test-data.json                     # 3 data-driven booking profiles incl. edge values
├── docs/
│   └── TEST_STRATEGY.md                           # Test strategy: scope, techniques, data, CI, limitations
├── package.json                                   # newman ^6, newman-reporter-htmlextra, npm scripts
├── package-lock.json
├── LICENSE                                        # MIT
└── .gitignore
```

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) 22.x (any current LTS works)
- Or [Docker](https://www.docker.com/), if you prefer not to install Node
- Or just the [Postman](https://www.postman.com/downloads/) app for interactive exploration

### Option 1 — Postman app

Import `collections/restful-booker.postman_collection.json` and `environments/production.postman_environment.json`, select the environment, and run the collection with the Collection Runner.

### Option 2 — Newman locally

```bash
npm ci
npm test
```

| Script | Description |
|---|---|
| `npm test` | Full suite against the hosted API (CLI reporter) |
| `npm run test:local` | Full suite against a local Docker instance (`localhost:3001`) |
| `npm run test:data` | Data-driven run — 3 iterations from `data/booking-test-data.json` |
| `npm run test:report` | Full suite with CLI + htmlextra + JUnit reporters → `reports/` |
| `npm run test:smoke` | Health Check folder only — fast availability probe |

### Option 3 — Docker (no Node install)

```bash
docker run -v $(pwd):/etc/newman -t postman/newman run \
  collections/restful-booker.postman_collection.json \
  -e environments/production.postman_environment.json
```

### Running the API itself locally

The hosted instance is shared and resets periodically. For a stable target:

```bash
docker run -d -p 3001:3001 mwinteringham/restfulbooker
npm run test:local
```

## Reports

- `npm run test:report` writes an **htmlextra** HTML dashboard (request/response detail, pass/fail breakdown, iteration view) and a **JUnit XML** file to `reports/`.
- In CI, every run uploads two artifacts — `newman-html-report` and `newman-junit-results` — retained for 30 days, plus a results table in the GitHub Actions step summary.
- Runs on `main` (pushes and the nightly cron) also publish the HTML report to **[GitHub Pages](https://qasimmahmood95.github.io/postman-api-automation/)**, so the latest results are one click away — including failed runs, which publish honestly rather than leaving a stale green dashboard.

## CI/CD

The **API Tests** workflow (`.github/workflows/api-tests.yml`) runs on:

- **Push / pull request** to `main`
- **Nightly cron** at 02:30 UTC — catches upstream API drift between commits
- **Manual dispatch** with an environment choice — `production` (hosted API) or `local`, which spins up Restful Booker in Docker on the runner and tests against it in full isolation

Pipeline: `npm ci` → full collection run with CLI + htmlextra + JUnit reporters → separate data-driven step (3 iterations) → artifact upload → step-summary table of results → HTML report published to GitHub Pages (main runs only).

The pipeline needs **no secrets** — Restful Booker's credentials are public practice values, so anyone can fork this repo and CI works immediately.

## Known API Quirks

Restful Booker is intentionally imperfect. The suite asserts its *real* behavior and flags each deviation in test comments, rather than writing conventional assertions that would fail against the actual API:

| Quirk | Convention says | Restful Booker does |
|---|---|---|
| `GET /ping` | 200 OK | **201 Created** |
| `DELETE /booking/:id` | 200 / 204 | **201 Created** |
| Invalid payload types | 400 Bad Request | **500 Internal Server Error** |

Treating these as documented, deliberately-asserted behavior keeps the suite honest: if the API is ever fixed, tests fail loudly and the assertions get updated consciously.

## License

[MIT](LICENSE) © Qasim Mahmood
