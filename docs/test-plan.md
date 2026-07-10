# Test Plan -- Restful-Booker API

## Scope

Functional and performance testing of the public [Restful-Booker](https://restful-booker.herokuapp.com)
demo API (auth, health check, and booking CRUD). Built as a QA/API-testing portfolio project, not as
a production regression suite.

Restful-Booker is deliberately non-standard in a few places (documented by its author as containing
"a bunch of bugs for you to explore"). Where a test in this suite hits one of those, it's called out
below rather than silently worked around -- finding and documenting that kind of thing is the point
of testing against this API in the first place.

## Test Strategy

- **Functional testing (Postman):** positive-path CRUD, negative/edge cases (invalid credentials,
  missing fields, non-existent IDs, missing auth), and a full create -> read -> update -> patch ->
  delete -> confirm-deleted chain using environment variables to pass state between requests.
- **Performance testing (JMeter):** 50 concurrent simulated users against `GET /ping` and
  `GET /booking`, 10 iterations each, checking response code and a 2-second response-time ceiling.
- **Automation:** the Postman collection runs on every push/PR via GitHub Actions (Newman), so a
  regression in the target API's contract would show up in CI, not just when run manually.

## Functional Test Cases

| ID | Endpoint | Scenario | Type | Expected Result |
|----|----------|----------|------|------------------|
| TC-01 | GET /ping | Health check | Positive | 201, responds within 2s |
| TC-02 | POST /auth | Valid admin credentials | Positive | 200, response contains `token` |
| TC-03 | POST /auth | Invalid password | Negative | 200 with `reason` field, no token issued |
| TC-04 | GET /booking | List all booking IDs | Positive | 200, array of `{bookingid}` objects |
| TC-05 | GET /booking?firstname=&lastname= | Filter with a name that shouldn't exist | Edge | 200, empty array |
| TC-06 | POST /booking | Create booking with full valid payload | Positive | 200, response echoes submitted data + new `bookingid` |
| TC-07 | POST /booking | Missing required fields (`lastname`, `bookingdates`) | Negative | Confirmed: 500 (see Findings -- this is a real defect, not a pass) |
| TC-08 | GET /booking/{id} | Fetch the booking just created | Positive | 200, fields match what was submitted |
| TC-09 | GET /booking/999999 | Fetch a non-existent ID | Negative | 404 |
| TC-10 | PUT /booking/{id} | Full update, valid auth | Positive | 200, fields reflect the update (see Findings -- Basic Auth required, not Cookie) |
| TC-11 | PUT /booking/{id} | Full update, no auth header | Negative | 403 |
| TC-12 | PATCH /booking/{id} | Partial update (single field), valid auth | Positive | 200, only the targeted field changes |
| TC-13 | DELETE /booking/{id} | Delete with valid auth cookie | Positive | 201 (Restful-Booker's documented delete response code) |
| TC-14 | GET /booking/{id} | Confirm the booking is gone post-delete | Negative | 404 |

## Performance Test Scenarios

| ID | Endpoint | Load Profile | Pass Criteria |
|----|----------|--------------|---------------|
| PERF-01 | GET /ping | 50 users, 10s ramp-up, 10 iterations | 0% error rate, p95 response time < 2000ms |
| PERF-02 | GET /booking | 50 users, 10s ramp-up, 10 iterations | 0% error rate, p95 response time < 2000ms |

## Performance Test Results (run 2026-07-10, 5:25-5:26 PM)

**Overall: did not meet pass criteria.** Reporting this as a miss, not rounding it to a pass --
2.2% error rate against a 0% target is a real result worth understanding, not a rough pass.

| Endpoint | Samples | Fail | Error % | Avg (ms) | Median (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Max (ms) | Throughput (req/s) | Apdex |
|----------|---------|------|---------|----------|--------------|----------|----------|----------|----------|---------------------|-------|
| Total | 1000 | 22 | 2.20% | 639.91 | 585.00 | 1229.80 | 1514.65 | 2411.54 | 2806 | 26.58 | 0.699 |
| GET /booking | 500 | 20 | 4.00% | 870.71 | 634.00 | 1464.10 | 1777.30 | 2648.90 | 2806 | 14.20 | 0.458 |
| GET /ping | 500 | 2 | 0.40% | 409.12 | 300.00 | 1107.30 | 1240.60 | 1819.01 | 2224 | 13.94 | 0.939 |

**What the numbers say:**

- **The two endpoints are not equally healthy.** `GET /ping` carried nearly all its load fine
  (0.40% errors, Apdex 0.939 -- Excellent). `GET /booking` accounted for 20 of the 22 total
  failures (91% of all errors) and scored 0.458 on Apdex -- solidly in the Unacceptable band.
  A single blended "2.2% error rate" headline would have hidden that this is really a
  `/booking`-specific problem, not a general capacity issue.
- **This connects directly to a functional-test finding, not just a load-test one.** TC-04
  (Findings Log below) already noted that `GET /booking` returns far more records than the
  documented 10 seed rows, because Restful-Booker is a shared public database that never
  resets during active testing. Under 50 concurrent users, that larger, growing payload is a
  plausible direct cause of both the higher error rate and the p95/p99 tail (1777ms / 2649ms)
  compared to `/ping`'s p95/p99 (1241ms / 1819ms) on effectively the same load profile.
- **The specific failures were response-time assertion trips, not connection or HTTP errors.**
  The JMeter Errors table shows entries like *"The operation lasted too long: it took 2806ms,
  but should not have lasted longer than 2000ms."* -- i.e. the requests succeeded and got a
  real response, they just came back slower than the 2-second threshold this test plan set.
  That's a meaningfully different finding than "the API returned 500s under load," and worth
  stating precisely rather than lumping it in as a generic failure.

## Findings Log

Use this section to record anything that surprised you when you actually ran the suite --
this is meant to be filled in from real runs, not assumed in advance.

| Date | Test Case | Observed vs Expected | Notes |
|------|-----------|----------------------|-------|
| 2026-07-10 | TC-11/TC-10, TC-12, TC-13 (PUT, PATCH, DELETE) | Cookie-based auth (`Cookie: token={{auth_token}}`, set from `POST /auth`) returned 403 despite a valid token | Root cause: Postman does not reliably send manually-set `Cookie` headers -- cookies are handled through Postman's own Cookie Jar, not the Headers tab, so the header configured in the collection never actually went out on the wire. Switched all three requests to Basic Auth (`admin` / `password123`), which Restful-Booker also documents as supported, and every request succeeded immediately after. Lesson: when an auth method fails unexpectedly in a tool, check whether the tool has special handling for that header type before assuming the credentials themselves are wrong. Fix moved to folder-level auth (Booking CRUD folder -> Authorization -> Basic Auth) so it isn't set per-request. |
| 2026-07-10 | TC-07 | Expected 400/500 (test written to accept either); observed 500 Internal Server Error | Real defect, not a pass-by-technicality: submitting `POST /booking` with `lastname` and `bookingdates` missing should return a clean 400 Bad Request. Instead the API throws a 500, meaning input isn't validated before the server tries to process it. Worth calling out specifically in an interview as "found a defect," not just "wrote a test." |
| 2026-07-10 | TC-01 | Expected 200 for a read-only health check; observed 201 Created (confirmed, matches Restful-Booker's documented behavior) | Non-standard but consistent -- 201 normally signals "a resource was created," which doesn't semantically fit a GET health check. Not a bug, just worth noting as a documented API quirk. |
| 2026-07-10 | TC-13 | Expected 201 (my best guess when the collection was built, unverified); observed 201 Created, confirmed | Guess was correct. Restful-Booker's DELETE returns 201 rather than the more conventional 200 or 204 No Content for a successful delete. |
| 2026-07-10 | TC-04 | Expected structure matched, but `GET /booking` returned far more than the documented 10 seed records | Restful-Booker is a shared public API -- other testers' data (and bots) persist in the same database until the 10-minute reset. Test was intentionally written to check structure (array of `{bookingid}` objects) rather than an exact count, so this didn't break anything, but it's a real characteristic of testing against a shared, non-isolated environment worth knowing going in. |
| 2026-07-10 | PERF-01 / PERF-02 | Pass criteria was 0% error rate and p95 < 2000ms for both endpoints; observed 2.2% overall error rate, and `GET /booking` p95 = 1777ms with a 2649ms p99 and 4.00% error rate (vs. `GET /ping` at 0.40% errors, Apdex 0.939) | Load test failed its own pass criteria -- reported as a miss, not adjusted after the fact. Failures were response-time assertion trips (requests exceeded the 2000ms threshold), not HTTP-level errors. `GET /booking` drove 20 of 22 total failures and is the clear bottleneck, plausibly linked to the larger-than-documented payload size already noted in TC-04. See full breakdown in Performance Test Results above. |


## How to Run

### Postman / Newman
```bash
npm install -g newman newman-reporter-htmlextra
newman run postman/RestfulBooker.postman_collection.json \
  -e postman/RestfulBooker.postman_environment.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export reports/newman-report.html
```
Or import both files into the Postman desktop app and run the collection from there first --
recommended before trusting the CI run, so you can see actual response bodies for the negative
cases and update the Findings Log above.

### JMeter
```bash
jmeter -n -t jmeter/restful-booker-load-test.jmx -l jmeter/reports/results.jtl -e -o jmeter/reports/html
```
`-o jmeter/reports/html` generates the HTML dashboard report (throughput, response time
percentiles, error rate) after the run completes.

### CI
Runs automatically on every push/PR to `main` via `.github/workflows/api-tests.yml`. The HTML
report is uploaded as a workflow artifact regardless of pass/fail.
