# What we built beyond the design submission

This lists everything in the repository that goes **beyond** [`docs/design/design_submission.md`](design/design_submission.md).
It was put together by reading the design, then the code: every route in `backend/src/routes.js`, every module
under `backend/src/modules/`, `ai/search.js` and its prompts, the migrations, all frontend pages, the scripts
and the workflows.

**Baseline: what the design promised (§3 MVP).** Availability search · AI natural-language search (en/hi) ·
10-minute TTL hold · 30 s expiry worker · confirm with mock payment · hotel + flight saga with compensation ·
cancellation with restock · idempotency keys · in-app load-test dashboard (p50/p95/p99) · web UI · Hindi + English.
None of those are listed below unless we went clearly further than the design described.

Each item gives what was built, where it lives, and what proves it.

---

## 1. Concurrency and correctness

| # | Extra | Design said | What we built | Where | Proof |
|---|---|---|---|---|---|
| 1.1 | **Atomic reserve inside one Postgres function** | Lock, check, insert, update from the app | `kognivera_create_holds()` runs lock → check → insert → update in **one database call**, so the row lock is held for microseconds instead of across network round trips. The Node transaction is kept as a switch (`HOLD_IMPL=sql\|js`) and both paths are covered by the same tests. | `data-model/migrations/002_create_holds_function.sql`, `backend/src/modules/booking/holds.js` | `tests/holds.test.js` |
| 1.2 | **Sold-out shield (flash-sale protection)** | — | Once a row is known to be full, this process answers "sold out" from memory for 300 ms. Single-row attempts queue per row **in memory** instead of on database locks. The shield can only reject early, never grant, and it clears the moment this process frees units. Load tests bypass it with `X-Bypass-Shield: 1` (only outside production) so the row lock is what gets tested. | `backend/src/modules/inventory/soldout.js` · `FAST_REJECT`, `SOLD_OUT_CACHE_MS` | `tests/holds.test.js` (the shield never blocks a real free-up) |
| 1.3 | **Multi-row atomic holds with one shared deadline** | One hold per room | One `POST /api/holds` can hold many rows at once: every night of a stay, a hotel plus a flight, or both legs of a connection. They are all held or none is, under **one `expires_at`**. Each row's key is derived as `<client key>#<i>`, so the whole request stays idempotent. A stay shorthand `{entity_type, entity_id, for_date, nights}` expands to one row per night. | `holds.js`, `modules/inventory/availability.js#resolveHoldItems` | `tests/holds.test.js` (multi-night atomic; trip shares one deadline) |
| 1.4 | **Idempotency misuse and in-flight detection** | Duplicate key returns the existing record | Reusing a key for a **different** request is refused with `422 idempotency_conflict`. A retry that arrives while the saga is still running gets `409 request_in_progress` + `Retry-After`. Retries are answered from the holds table **before** taking any lock. Responses carry an `Idempotent-Replayed` header. | `holds.js#replayFrom`, `bookings.js#replay`, `app.js` | `tests/holds.test.js`, `tests/bookings.test.js`, `tests/api.test.js` |
| 1.5 | **Deadlock retry, and locking before insert** | Fixed lock order only | `withTx` retries deadlocks (`40P01`) with jitter. Rows are locked **before** a hold is inserted, because the FK on `holds.inventory_id` takes `FOR KEY SHARE`, and insert-then-lock can deadlock two requests against each other. | `backend/src/db.js`, `holds.js` header comment | `tests/holds.test.js` (opposite-order multi-row holds never deadlock) |
| 1.6 | **Late confirm refused at confirm time** | The worker releases expired holds | The saga compares `expires_at` itself, so a hold past its deadline is dead **even if the sweep hasn't run yet**. The worker only returns capacity. | `bookings.js#confirmLine` | `tests/bookings.test.js` (expired hold → saga rolls back) |
| 1.7 | **Compensation retries and a visible failure state** | Listed as a known limitation | Each compensation is its own idempotent transaction, retried 3× with backoff. If it still fails, the booking is parked as `partially_confirmed` and the API returns `500 compensation_incomplete`. The failure is visible instead of silent. | `bookings.js#compensate` | `tests/bookings.test.js` (all three failure modes compensate) |
| 1.8 | **Safety-net counter** | The DB CHECK is the hard backstop | Every time the canonical `CHECK (booked+held<=total)` rejects a write, it is **counted** (`metrics.safety_net_hits`, `GET /api/metrics`). The load test asserts the count stays **0**, because a correct application never lets the database refuse. Removing `FOR UPDATE` makes this check fail. | `backend/src/errors.js#fromPgError`, `modules/loadtest/engine.js` | `db_check_never_fired` in every load-test verdict |
| 1.9 | **Four invariant queries, not one** | "DB invariant query returning zero violations" | `oversold`, `negative`, `held_drift` (held units ≠ Σ active holds) and `booked_drift` (booked units ≠ Σ confirmed lines). The two drift checks prove **no unit was ever lost or double-counted**, which is stronger than "not oversold". | `backend/src/modules/invariants.js` · `GET /api/invariants` · `npm run invariants` | every test suite and load test |
| 1.10 | **Clean overload behaviour** | `lock_timeout` → "sold out" | Lock timeout, a deadlock after its retries, **pool exhaustion** (a plain client error with no SQLSTATE) and **database unreachable** all map to `503 contention_timeout` + `Retry-After: 1`, never a bare 500. The HTTP accept backlog is raised to 2048 so a 500-way burst isn't refused at TCP. | `errors.js`, `db.js` (`PG_POOL_CONNECT_TIMEOUT_MS`), `server.js` | `tests/errors.test.js` · 2-connection pool vs 500 requests: 461 × 503, 0 × 500 |
| 1.11 | **Property booking rules enforced** | — | `min_stay_nights` and `closed_to_arrival` from `inventory_calendar` are enforced on hold (`422 constraint_infeasible`) and filtered out of search. | `availability.js`, `search.js` | No dedicated test yet |
| 1.12 | **Horizontal scaling (cluster mode)** | Load balancer explicitly **out of scope** | `npm run start:cluster` starts N round-robin worker processes. It is safe because correctness lives in Postgres, and it has been measured at up to 8 processes with zero oversell. Load-test progress is mirrored to `load_test_runs.snapshot` so **any** worker can answer a poll. | `backend/src/cluster.js`, `migrations/003_load_test_snapshot.sql` | `backend/README.md` (measured: correct at every size, not faster on a hot row) |
| 1.13 | **Strict request validation** | — | zod schemas with `.strict()` (unknown fields rejected), bounds of 1–20 units per line, 1–60 lines per request, TTL of 5–1800 s, ISO-4217 currencies and an idempotency-key format, all returned as field-level `400` errors. | `backend/src/validation.js` | `tests/api.test.js` |
| 1.14 | **Bilingual API errors** | Hindi for UI labels | Every API error `message` is localised server-side (`?lang=hi` or `Accept-Language: hi`), using a stable `code` catalogue of 19 errors. This includes the saga rollback message. | `backend/src/errors.js` | `tests/api.test.js` (Hindi saga-failure message) |

## 2. Load testing and proof

The design promised an in-app dashboard showing counts and p50/p95/p99. What we built on top of that:

| # | Extra | What it adds | Where |
|---|---|---|---|
| 2.1 | **Verdict read from the database, 6 checks** | `no_oversell`, `exactly_free_units_granted`, `no_request_granted_twice`, `counters_reconcile`, `db_check_never_fired` and `no_transport_or_server_errors`. It checks the tables after the race rather than trusting the responses. | `modules/loadtest/engine.js` |
| 2.2 | **Duplicate-key mode** | `duplicate_factor` 1–5 sends every logical request N times **simultaneously with the same key**. This proves retries never double-book, and does it under contention. | engine, Load test page |
| 2.3 | **Evidence the contention reached Postgres** | `pg_stat_activity` is sampled every 5 ms during the race and reported as `peak_db_sessions_blocked_on_row_lock`. It proves requests actually queued on the row lock. | engine |
| 2.4 | **Richer run data** | Live timeline (every 100 ms), latency histogram, throughput, transport retries, `api` vs `direct` mode, automatic cleanup that restores the row, and every attempt persisted to `load_test_results`. | engine, `load_test_runs` / `load_test_results` |
| 2.5 | **CLI race** | `npm run loadtest -- --requests 500` runs from a separate process and exits non-zero on any failed check. | `backend/scripts/loadtest.js` |
| 2.6 | **Mixed confirm / abandon scenario** | Winners of the race confirm and the rest let their hold expire. It then asserts `booked + held + available == total` against figures derived from **the individual hold records**, not the row's own counters. It also checks that no expired hold still holds stock and samples zero oversell throughout. It **fails if the expiry sweep is disabled**, so it actually tests something. | `backend/scripts/mixed-loadtest.mjs` · `npm run loadtest:mixed` |
| 2.7 | **k6** | An industry tool. Every virtual user sends a different `X-User-Id`, and `teardown()` checks `/api/invariants` directly. There is also a ramping read-only script for a k6 HTML report, and a saved report. | `backend/scripts/k6-loadtest.js`, `k6-report-demo.js`, `docs/evidence/k6-report.html` |
| 2.8 | **Distributed runs from separate machines** | Three GitHub Actions workflows fire from a matrix of runner VMs, then verify against the target's database. **Unique keys** prove serialization. The **same key from every machine** proves exactly one `hold_id` is ever returned. | `.github/workflows/distributed-load-test.yml`, `distributed-idempotency-test.yml`, `backend/scripts/gh-*.mjs` |
| 2.9 | **Dropped-response retry** | The request is sent and the client aborts before the response arrives, then retries with the same key. It reports honestly whether the server had already finished, and asserts exactly one hold per run. | `backend/scripts/simulate-network-retry.mjs`, `.github/workflows/network-retry-test.yml` |
| 2.10 | **System Visualizer** | Animated race, saga-failure and duplicate-retry scenarios. **Simulated** mode replays a scripted timeline. **Live backend** mode drives the real API (a real load test, a real forced saga failure, a real duplicate confirm) and shows what the server returned. | `frontend/src/pages/VisualizerPage.jsx`, `frontend/src/viz/live.js`, `viz/scenarios.js` |
| 2.11 | **60 automated tests against real Postgres** | Races (200-way, multi-unit, 60 hold-then-book), simultaneous retries, key reuse, deadlock order, expiry, all saga failure modes, 12 simultaneous cancels, the HTTP contract, sessions, one-stop flights, error mapping and AI grounding. | `tests/*.test.js` |
| 2.12 | **Supporting tools** | `check-gemini.js` checks the live AI path end to end. The Python `tests/load_test.py` loads directly against the database. `data-model/tools/validate_postgres.py` checks conformance plus invariants. | `backend/scripts/`, `tests/`, `data-model/tools/` |

## 3. Product features the design left out of scope

| # | Extra | Design said | What we built | Where |
|---|---|---|---|---|
| 3.1 | **Mock login with 10 personas** | "User authentication / sessions — out of scope; we use a demo user" | A login screen lists 10 real seeded travellers, chosen by a fixed rule: two INR users, then one per other home currency, so switching user also shows localised prices. There is also an **operator** role. The identity is sent as `X-User-Id` and checked against `users`. Each tab is its own session. Travellers **cannot read or act on each other's** holds and bookings (`403`), and the operator cannot act as a traveller. Scripts that send no header fall back to the demo user. This is demo identity, not real auth. | `backend/src/modules/session.js`, `frontend/src/session.jsx`, `pages/LoginPage.jsx` · `tests/session.test.js` |
| 3.2 | **Operations dashboard** | "Admin panel — no demo value" | Operator-only and refreshed every 2 s: an invariants hero (0 violations), holds and bookings by status (plus the last hour), inventory totals, a **watched room** with live counts, and an activity feed of who held, booked or was **rejected**. Rejected attempts leave no DB row, so they are kept in an in-memory ring buffer. **Reset demo** releases the personas' holds and cancels their bookings, leaving seed data untouched. There is a deep link to the watched room for the two-user scenario. | `backend/src/modules/ops.js`, `pages/OpsPage.jsx` · `GET /api/ops/summary`, `POST /api/ops/reset-demo` |
| 3.3 | **Flight search UI** | "Flight search UI — out of scope; flights only as the saga's second item" | A full Flights tab: origin/date pickers fed by `GET /api/flights/routes` (only routes with seats), a search button that is disabled and says why when a route or date has no seats, and results with fare, baggage and seat count. | `pages/HomePage.jsx`, `pages/SearchPage.jsx`, `search.js#searchFlights` |
| 3.4 | **One-stop flight connections** | — | Plain SQL pairs two flights where the second leaves the **same airport** 60–360 min after the first lands. Both legs are one trip item and are **held in one atomic request**, so a sold-out second leg leaves nothing held. Routes reachable only with a stop are offered too. The migration `004_hub_flights.sql` adds a daily schedule through New Delhi and Mumbai (extra rows only), because the seed has too few connecting flights. | `search.js#searchConnections`, `flightRoutes` · `tests/connections.test.js` |
| 3.5 | **Trip cart, then one Reserve** | "Hold Room" puts an immediate hold per room | Adding to the trip holds **nothing**, and the item is a draft. One **Reserve** holds everything together under one countdown. Changing a reserved trip releases the reservation, so items can never carry different deadlines. A sold-out item fails the whole request and is named. The client polls the server every 7 s for holds released elsewhere, and the trip is stored per user. | `frontend/src/trip.jsx`, `pages/HoldPage.jsx` |
| 3.6 | **Demo controls** | Fault injection not mentioned | A header menu (only when the backend is not in production) sets a short hold time (15 s / 60 s) and **simulates a checkout failure**: the flight sells out, the hotel sells out, or payment is declined. Each setting applies to one checkout and is never persisted, so it can't leak into a real booking. Server side, `simulate_failure` is refused unless `ALLOW_FAULT_INJECTION`. | `App.jsx#DemoMenu`, `bookings.js` |
| 3.7 | **Card / UPI payment form** | "Confirm/pay modal" | Client-side validation (Luhn card check, expiry, CVV, UPI id format) and **Fill test details**. Card fields are never sent; only the method is. | `pages/HoldPage.jsx` |
| 3.8 | **Idempotency on screen** | "Clicking Confirm again returns the same booking" | The confirmation page has **Retry the same request**, which re-sends the exact stored `{key, body}` and shows "Same booking returned. Nothing was booked twice." A rolled-back booking shows its compensated lines struck through, with line-by-line status. | `pages/ConfirmationPage.jsx` |
| 3.9 | **Rate plans priced into the booking** | Rate plans used for search filters | The chosen `rate_plan_id` goes into `POST /api/bookings`. Its `price_delta` is added per unit, validated to belong to that room type and currency, and shown in the line title. | `bookings.js#prepare` · `tests/bookings.test.js` |
| 3.10 | **Rooms stepper and sold-out rooms** | — | The hotel page lists fully booked room types as such (`include_sold_out`) and caps the Rooms stepper at the free count. The hold requests that many units. | `pages/HotelPage.jsx` |
| 3.11 | **Currency tools** | Display conversion | `GET /api/currencies` and `GET /api/fx?from&to&amount`, plus a header currency switcher that defaults to each user's home currency. A search budget stays in **the currency it was stated in** (`budget_currency`), separate from the display currency. | `routes.js`, `fx.js`, `search.js` |
| 3.12 | **Integer-cent money in the browser** | `decimal.js` on the server | The frontend does price and tax arithmetic in **BigInt minor units** and never uses floats. It mirrors the server's 12% tax rule, and the server stays authoritative. | `frontend/src/lib/money.js` |
| 3.13 | **Self-configuring UI** | — | `GET /api/meta` returns the inventory date window, a default check-in, the signed-in user, whether AI is on, whether demo controls are allowed, and the hold TTL. `GET /api/cities` flags which cities actually have rooms (43 of 60). | `routes.js` |
| 3.14 | **One process serves everything** | Separate frontend / backend | `npm start` serves the API **and** the built web app, with a History-API router so every page URL can be refreshed. | `backend/src/app.js`, `frontend/src/router.jsx` |
| 3.15 | **Full Hindi UI with a key check** | "Key surfaces" in Hindi | 485 keys in both `en.json` and `hi.json`. `npm run check:i18n` fails if a key or placeholder is missing. | `frontend/src/locales/`, `frontend/scripts/check-i18n.mjs` |

## 4. AI beyond the design

The design described Gemini function calling for hotel search, a summary, and a structured form plus pre-cached demo
queries as the fallback. Beyond that:

| # | Extra | What we built | Where |
|---|---|---|---|
| 4.1 | **Flight NL search with its own schema** | The design mentioned flights in prose, but its extraction schema was hotel-only. We added a separate `search_flights` tool and prompt that parses origin/destination **roles** ("from X to Y"), dates and seats, and runs the same SQL as the flight form. | `ai/prompts/search_flights.tool.json`, `search_flights_system.md`, `ai/search.js#aiFlightSearch` |
| 4.2 | **Model fallback chain** | The configured models (`GEMINI_MODEL` + `GEMINI_FALLBACK_MODELS`) are tried in turn with a **6 s timeout each**, so a busy, retired or slow model doesn't take search down. Calls go over REST (`fetch`) rather than the SDK. | `ai/search.js#callGemini` |
| 4.3 | **Offline heuristic parser** | An English fallback for **both** hotels and flights: cities (longest match, aliases like Goa → Panaji), date ranges, nights, rooms, adults, stars, budget + currency symbol, breakfast/refundable, and from/to roles. Search keeps working with no API key. | `ai/search.js#parseHeuristic`, `parseFlightHeuristic` · `tests/ai.test.js` |
| 4.4 | **Ask instead of guess** | A missing or unknown city returns `needs_clarification`. A missing flight origin comes back with the **real origins** that fly to that destination. Zero results come with a reason (`city_has_no_inventory` + the cities that do have rooms, `no_match_for_filters`, `no_route`, or `no_flights_on_date` + the available dates). | `ai/search.js` · `tests/ai.test.js` |
| 4.5 | **Constrained to real cities** | The prompt receives the live list of active city names from the database, and the answer is rejected if the city isn't one of them. | `search_system.md`, `ai/search.js` |
| 4.6 | **Budget currency inference** | A budget with no currency is inferred from the text (₹/रुपये/$/€/£), defaulting to INR, so a ₹5000 budget is never compared against a foreign hotel's native price. | `ai/search.js#budgetCurrency` |
| 4.7 | **Transparency and audit** | Every answer reports `parser: cache \| gemini \| heuristic` and the model used, and the UI shows it as a badge. `search_logs` records the parser and latency as well as the query. Successful Gemini parses are cached. | `ai/search.js`, `pages/SearchPage.jsx`, `migrations/001_additions.sql` |

## 5. Data model beyond the design

The design planned three new tables. All additions below are additive under rule R1, and nothing canonical was changed.

| Addition | Why |
|---|---|
| `booking_items.hold_id` + index | Ties each booking line to the exact hold it consumed, so compensation is exact even when two lines hit the same row. |
| `CHECK (booked_units >= 0 AND held_units >= 0)` | The canonical CHECK can't see a counter going negative, which could hide an oversell. |
| `idx_holds_active_expiry` (partial) | The expiry sweep scans only live holds. |
| `kognivera_create_holds()` | See 1.1. |
| `load_test_runs.snapshot` | See 1.12. |
| Extra columns in `load_test_runs` (mode, duplicate_factor, units_per_request, initial_free, expected_successes, sold_out, errors, oversold, max_ms, duration_ms, throughput_rps, verdict) and `load_test_results` (attempt_no, http_status, error_code, hold_id) | These store the full verdict and per-attempt detail rather than only aggregate counts. |
| `search_logs.parser`, `search_logs.latency_ms` | Records who answered and how fast. |
| `004_hub_flights.sql` (rows only, `flt_h`/`far_h`/`inv_h` ids) | Supply for one-stop itineraries (3.4). |

## 6. Design limitations we closed

| Limitation (design §12) | Status | How |
|---|---|---|
| Expiry worker is single-instance | **Closed** | `pg_try_advisory_xact_lock` means a second instance skips its turn. The sweep uses `FOR UPDATE SKIP LOCKED` in batches of 500. (`holds.js#expireHolds`, `workers/expiry.js`) |
| Compensation itself failing | **Partly closed** | Retried 3× and parked visibly as `partially_confirmed` (1.7). There is still no background recovery worker. |
| No saga orchestrator persistence | **Partly closed** | Each saga step commits on its own and its state is in `bookings` / `booking_items` / `payments`, so a crash leaves a readable state. A restarted process does not resume it automatically. |

---

## For context: deviations and gaps (not extras)

These aren't features. They are listed so the comparison is complete and nobody claims them by mistake.

- **Changed from the design:** no Knex (plain SQL migrations + `pg`) · Gemini over REST instead of `@google/generative-ai` · local Docker Postgres instead of Railway · a light UI with a dark ops theme instead of an all-dark theme · lock contention returns `503 contention_timeout`, not "sold out".
- **Planned but not built:** the 20-query AI accuracy eval (10 en + 10 hi) with a ≥ 90% target, since `tests/ai.test.js` covers grounding and parsing but isn't that eval · AI translation of hotel descriptions to Hindi · a `saga_log` table · rate-plan cancellation penalties (cancellation refunds in full).
- **Housekeeping:** `frontend/src/pages/TripPage.jsx` isn't routed or imported anywhere. `/hold` renders `HoldPage.jsx`.
