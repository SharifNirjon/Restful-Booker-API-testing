# API Test Portfolio -- Restful-Booker

A small QA/API-testing portfolio project: functional tests (Postman/Newman), performance tests
(JMeter), and CI automation (GitHub Actions) against the public
[Restful-Booker](https://restful-booker.herokuapp.com) demo API.

## Why this API

Restful-Booker is built specifically for practicing API testing -- it has auth, full CRUD, and
(by its own documentation) a few intentionally non-standard behaviors to find. That last part
matters: the goal here isn't just "run some requests," it's demonstrating the habit of noticing
and documenting when a system doesn't behave the way you'd expect. See `docs/test-plan.md` ->
Findings Log.

## Structure

```
api-test-portfolio/
├── postman/
│   ├── RestfulBooker.postman_collection.json   # functional tests: auth, CRUD, negative cases
│   └── RestfulBooker.postman_environment.json  # base_url, auth_token, booking_id variables
├── jmeter/
│   └── restful-booker-load-test.jmx            # 50-user concurrent load test
├── .github/workflows/api-tests.yml             # runs the Postman collection via Newman on push/PR
└── docs/test-plan.md                           # test case table, performance scenarios, findings log
```

## Quick Start

**Functional tests:**
```bash
npm install -g newman newman-reporter-htmlextra
newman run postman/RestfulBooker.postman_collection.json \
  -e postman/RestfulBooker.postman_environment.json \
  --reporters cli,htmlextra --reporter-htmlextra-export reports/newman-report.html
```

**Load test:**
```bash
jmeter -n -t jmeter/restful-booker-load-test.jmx -l results.jtl -e -o jmeter/reports/html
```

**Or** import both Postman files into the Postman app and run interactively -- recommended for
your first pass, so you can see actual response bodies and fill in the Findings Log in
`docs/test-plan.md` before wiring things into CI.

## What this covers

- Auth flow (valid + invalid credentials)
- Full CRUD chain on a single resource, with state passed between requests via environment
  variables (create -> read -> update -> patch -> delete -> confirm-deleted)
- Negative/edge cases: missing fields, non-existent IDs, unauthenticated writes
- Load behavior under concurrent users with response-time thresholds
- CI wiring so functional tests run automatically on every push/PR
