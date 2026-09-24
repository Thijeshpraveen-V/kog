# APS-05 — Distributed Booking & Inventory System

## Kognivera · Team *--rebase*

A booking and inventory service that **never oversells under concurrency**, and a load test that proves it.

---

## 1. Team & Problem Statement

| | |
|---|---|
| **Team Name** | *--rebase* |
| **Problem Statement** | **APS-05 — Distributed Booking & Inventory System** |
| **Theme** | Travel & Tourism |
| **Full statement** | [`docs/problem-statement/ps_description.md`](docs/problem-statement/ps_description.md) |

| Member | Role |
|--------|------|
| **Thijesh** | Backend Lead: inventory service, concurrency logic, hold/booking APIs, saga pattern |
| **Tejas** | Backend + AI: Gemini integration, natural-language search, multilingual, API scaffold |
| **Yogita** | Frontend Lead: React UI, search interface, booking flow, load-test dashboard |
| **Neha** | Data & QA: database setup, data loading, load-test engine, integration testing |

**The challenge in one line.** When 500 people race for the last 3 rooms, exactly 3 succeed, retries never
double-book, and a failed multi-item booking rolls back cleanly. The invariant we defend is
`booked_units + held_units ≤ total_units`, which must hold at every instant and under any load.

---

## 2. What We Built

### Against the PS's MVP checklist

- **Availability search with a TTL hold.** You can search hotels and flights with the form or in natural language (English/Hindi). Items go into **My trip**, where nothing is held yet. One **Reserve** then holds the whole trip atomically, with one server-clamped TTL (5 s – 30 min) and one live countdown.
- **Confirm with payment; release on timeout.** `POST /api/bookings` captures a mock payment. A 30 s background sweep releases abandoned holds. A late confirm is refused at confirm time, whether or not the sweep has run.
- **No overbooking under concurrent requests.** Every reservation is one transaction: `SELECT … FOR UPDATE` in ascending id order, check, then update, all inside a single Postgres function. The database `CHECK` is only a safety net, and the tests assert it **never fires**.
- **Idempotent booking API.** Every write requires an `Idempotency-Key` and uses `INSERT … ON CONFLICT DO NOTHING`. A retry replays the original result. Reusing a key for a different request is refused.
- **Multi-item saga with compensation; cancellation with restock; localised currency.** A hotel and a flight go into one booking. If any line or the payment fails, every confirmed line is compensated. Cancelling refunds the booking and restocks in one transaction. Prices are converted through the dated `fx_rates` table.
- **The standout deliverable, a load test showing zero oversell.** It runs in the app, from the CLI, with k6, and from separate machines via GitHub Actions. Every verdict is read from the **database**, not from the HTTP responses.

### Also in the build

| Area | What we built |
|------|---------------|
| **Concurrency** | The atomic reserve runs inside one Postgres function (`kognivera_create_holds`). An in-memory **sold-out shield** answers requests for full rows without touching the database lock. Deadlocks are retried. Overload (lock timeout, exhausted connection pool, database down) returns `503 + Retry-After`, never a bare 500. **Cluster mode** runs several processes. |
| **Proof** | A 6-check verdict read from the database. A **duplicate-key** load mode. A sampled count of database sessions blocked on the row lock. A mixed confirm/abandon closing-balance test. k6. Three distributed GitHub Actions workflows. A dropped-response retry test. A **live System Visualizer**. 60 automated tests. |
| **Product** | **Mock login** with 10 personas and an operator role. An **Operations dashboard**. A full **flight search UI**. A trip cart followed by one atomic Reserve. Demo controls for fault injection. A Card/UPI form. A **Retry the same request** button. |
| **One-stop flights** | Travellers can book a flight **even when no direct flight connects two cities**. If a route exists through an intermediate city, the search offers it under **One-stop options** (for example Bengaluru → New Delhi → Jaipur). The second leg must leave the **same airport** the first lands at, 60 min to 6 h later. Both legs become one trip item and are **held in one atomic request under one timer**: a sold-out second leg leaves nothing held, and a connection can never be half-booked or oversold. Routes reachable only with a stop are also listed in the origin/date pickers. It is plain SQL over the same inventory, with no AI involved. |
| **AI** | Flight natural-language search. A multi-model fallback chain. An offline heuristic parser for hotels and flights. The app asks for a missing city or origin instead of guessing, and explains empty results. Budget currency is inferred from the text. The parser used is shown and logged. |
| **Resilience** | The expiry worker is safe with multiple instances (advisory lock). Failed compensation is retried and left in a visible `partially_confirmed` state. |

---

## 3. Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     FRONTEND — React 18 + Vite  (en / हिन्दी)              │
│                                                                          │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────────┐  │
│  │ Login   │ │ Search   │ │ My trip  │ │ My       │ │ Load test       │  │
│  │ (mock   │ │ hotels + │ │ Reserve  │ │ bookings │ │ System          │  │
│  │ personas│ │ flights, │ │ → Pay,   │ │ cancel   │ │ Visualizer      │  │
│  │ + ops)  │ │ AI bar   │ │ countdown│ │          │ │ Ops dashboard   │  │
│  └─────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────────────┘  │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ REST /api/* (JSON) · X-User-Id · Idempotency-Key
                                │ (built UI is served by the same process)
┌───────────────────────────────▼──────────────────────────────────────────┐
│               BACKEND — Node 20 + Express 5 (modular monolith)           │
│   routes → zod validation → session (mock login) → modules               │
│                                                                          │
│  ┌──────────────┐ ┌────────────────┐ ┌──────────────┐ ┌───────────────┐  │
│  │ Inventory    │ │ Booking        │ │ AI search    │ │ Load-test     │  │
│  │ • search     │ │ • createHold   │ │ (ai/)        │ │ engine        │  │
│  │ • connections│ │ • confirm saga │ │ • Gemini fn  │ │ • race N reqs │  │
│  │ • sold-out   │ │ • compensate   │ │   calling    │ │ • 6-check     │  │
│  │   shield     │ │ • cancel       │ │ • heuristic  │ │   DB verdict  │  │
│  └──────────────┘ └────────────────┘ └──────────────┘ └───────────────┘  │
│  ┌────────────────────────────┐ ┌─────────────────────────────────────┐  │
│  │ Payment (mock, outside txn)│ │ Ops · Invariants · Metrics          │  │
│  └────────────────────────────┘ └─────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ Expiry worker — every 30 s, pg_try_advisory_xact_lock (multi-safe) │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│   Optional: npm run start:cluster → N round-robin worker processes       │
└───────────────────────────────┬───────────────────────────┬──────────────┘
                                │ pg.Pool (max 20)          │ HTTPS
┌───────────────────────────────▼──────────────┐  ┌─────────▼────────────┐
│           PostgreSQL 16 (Docker)             │  │ Google Gemini        │
│  • SELECT … FOR UPDATE, ascending id order   │  │ function calling,    │
│  • kognivera_create_holds() — atomic reserve │  │ 3 models tried in    │
│  • UNIQUE idempotency_key on holds/bookings/ │  │ turn, 6 s timeout    │
│    payments                                  │  └──────────────────────┘
│  • CHECK booked+held ≤ total  (safety net)   │
│  • CHECK booked ≥ 0, held ≥ 0 (added)        │
│  • 41,855 seeded rows + our additive tables  │
└──────────────────────────────────────────────┘
```

### Booking flow (as built)

```
TRAVELLER                     BACKEND                                   POSTGRES
   │ Search (form or AI)        │ read-only snapshot: free = total−booked−held │
   │ Add to trip (draft)        │ nothing held yet                             │
   │ Reserve ──────────────────▶│ ONE call: kognivera_create_holds()           │
   │   (Idempotency-Key)        │   lock rows ASC → check → INSERT holds       │
   │                            │   ON CONFLICT DO NOTHING → held_units += n   │
   │ ◀── holds, one expires_at  │                                              │
   │ Pay now ──────────────────▶│ claim booking (pending) ON CONFLICT          │
   │   (Idempotency-Key)        │ per line, own txn: hold → booked             │
   │                            │ mock payment (no lock held) → finalise       │
   │                            │ on failure: compensate every line, restock   │
   │ ◀── confirmed | rolled_back│                                              │
   │ Cancel ───────────────────▶│ one txn: booked_units −= n, refund, R8       │
   │                            │ expiry worker (30 s): expire + release held  │
```

### Why these choices

| Decision | Why |
|----------|-----|
| **Modular monolith** | Shared transactions for the saga, one deploy target, and easy debugging under load. The module boundaries are ready to be split out later. |
| **`READ COMMITTED` + explicit row locks** | Conflicts are on known rows, so `FOR UPDATE` is enough and avoids the retry churn `SERIALIZABLE` would add. |
| **Fixed lock order (ascending `inventory_id`)** | Two multi-row requests can never lock the same rows in opposite order, so they can't deadlock. |
| **Lock before insert** | The FK `holds.inventory_id` takes `FOR KEY SHARE`. Inserting first and then locking could deadlock two requests against each other. |
| **No I/O inside a locked transaction** | The mock payment runs between saga steps, so a slow gateway never blocks the inventory row. |
| **Reserve in one Postgres function** | The row lock is held for microseconds instead of across network round trips. `HOLD_IMPL=js` keeps the Node version for comparison. |

More detail: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) · measured numbers: [`backend/README.md`](backend/README.md).

---

## 4. Data Model

We use the **canonical APS-05 model unchanged** (`data-model/schema.sql`, 20 seed CSVs, **41,855 rows**). No
canonical table or column is renamed, dropped or repurposed (rule R1). The APS-05 data pack identifies its tables by name, so they are listed by name here.

### Canonical Tables We Use (16 of 20)

| Table | How we use it |
|-------|---------------|
| **`inventory_calendar`** | **The contended resource.** Every hold, confirm, cancel and expiry changes it under a row lock. |
| **`holds`** | TTL reservations. The UNIQUE `idempotency_key` gives idempotent inserts. Expired and released rows are kept (R8). |
| **`bookings`** | Saga header: `pending → confirmed \| failed \| partially_confirmed \| cancelled`. |
| **`booking_items`** | One line per hold. `status='compensated'` + `compensated_at` make a rollback auditable. |
| **`payments`** | Mock payment: `initiated → captured → refunded`, or `failed` with a `failure_code`. |
| **`hotels`**, **`hotel_room_types`**, **`hotel_rate_plans`**, **`cities`** | Hotel search. The room type is the bookable unit, and a rate plan's `price_delta` is priced into the booking. |
| **`flights`**, **`flight_fares`**, **`airports`**, **`airlines`** | Flight search and one-stop connections. `flight_fares` is the second `entity_type`. |
| **`users`** | The 10 login personas, and ownership of holds and bookings. |
| **`currencies`**, **`fx_rates`** | Localised currency: `DECIMAL(12,2)` + ISO-4217, pivoting through INR on dated rates, and respecting `minor_unit_exponent` for display. |

`itineraries`, `trips`, `languages` and `countries` are loaded and kept intact but not used by our code.

### What We Added (additive, R1)

Applied by `npm run migrate` from `data-model/migrations/`, idempotent:

| Addition | Type | Purpose |
|----------|------|---------|
| `load_test_runs` | new table | One row per run: config, counts, p50/p95/p99, throughput, the full verdict JSON, `oversold`. |
| `load_test_results` | new table | One row per attempt: status, latency, HTTP status, error code, hold id. |
| `search_logs` | new table | Every AI search: query, BCP-47 language, parser used, parsed params, result count, latency. |
| `booking_items.hold_id` (+ index) | new column | Ties a line to the exact hold it consumed, so compensation is exact. |
| `load_test_runs.snapshot` | new column | Lets any cluster worker answer a load-test poll. |
| `CHECK (booked_units ≥ 0 AND held_units ≥ 0)` | new constraint | The canonical CHECK can't see a counter going negative. |
| `idx_holds_active_expiry` | partial index | The expiry sweep scans only live holds. |
| `kognivera_create_holds()` | function | Lock → check → insert → update in one call. |
| `004_hub_flights.sql` | rows only | A daily schedule through New Delhi and Mumbai so one-stop itineraries can be demoed. |

Full mapping and where each boundary rule is enforced and tested: [`data-model/DATA_MODEL.md`](data-model/DATA_MODEL.md).

---

## 5. AI Features

### Natural-Language Availability Search (hotels and flights, English and Hindi)

| | |
|---|---|
| **What it does** | A query like *"3-star hotel in Jaipur for 2 adults, Oct 10-12, under ₹5000"*, *"जयपुर में 2 रातों के लिए होटल, ₹5000 से कम"* or *"Bengaluru to Jaipur on Nov 4, 2 seats"* returns bookable results. |
| **Mechanism** | Google **Gemini function calling** over REST. The tool schemas are `ai/prompts/search_hotels.tool.json` and `search_flights.tool.json`, with system prompts `search_system.md` and `search_flights_system.md`. The configured models are tried in turn (`GEMINI_MODEL` + `GEMINI_FALLBACK_MODELS`), 6 s timeout each. |
| **How it is grounded** | The model **only fills search parameters**, and results come from the same SQL the form uses. The prompt receives the **live list of cities** from the database, and an answer naming any other city is rejected. Every result is a real `inventory_calendar` row with live availability. |
| **When unsure** | It doesn't guess. A missing city returns `needs_clarification`. A missing flight origin comes back with the **real origins** for that destination. Zero results come with a reason (the city has no inventory, no match for the filters, no route, or no flights on that date) and alternatives. |
| **Budget** | Currency is inferred from the text (₹ / रुपये / $ / € / £, default INR). A budget stays in the currency it was stated in, separate from the display currency. |

### Comparison Summary

| | |
|---|---|
| **Mechanism** | Gemini text generation (`ai/prompts/summarise.md`): two sentences in the query's language. |
| **How it is grounded** | It receives **only the top 3 result rows** and is told to "use only this data". If it fails, the search still succeeds without a summary. |

### Fallbacks and Transparency

| Order | Parser | When |
|-------|--------|------|
| 1 | **Cache** | The five demo queries are pre-seeded (`seedDemoQueries`), and successful Gemini parses are cached. |
| 2 | **Gemini** | `GEMINI_API_KEY` is set and one of the models answers. |
| 3 | **Heuristic** | An offline English parser for hotels and flights: cities + aliases (Goa → Panaji), date ranges, guests, stars, budget, preferences, and from/to roles. Free-text Hindi needs the key. |

Every response reports `parser: cache | gemini | heuristic` and the model used. The UI shows it as a badge, and it
is logged to `search_logs`. **No AI is on the booking path.** Holds, the saga and one-stop connections are plain SQL.

Code: [`ai/search.js`](ai/search.js) · tests: `tests/ai.test.js`.

---

## 6. Run It Locally

**Prerequisites:** Node ≥ 20, Docker Desktop, Python 3 (for the seed loader).

| Step | Command | What it does |
|------|---------|--------------|
| **1. Database** | `docker compose up -d` | Postgres 16 on `localhost:5433` (db `kognivera`, user/pass `postgres`/`postgres`). |
| **2. Seed tools** | `pip install -r data-model/tools/requirements.txt` | psycopg for the loaders. |
| **3. Schema** | `python data-model/tools/apply_schema.py --schema data-model/schema.sql` | Canonical DDL. |
| **4. Seed** | `python data-model/tools/load_data.py --csv-dir data-model/seed/csv` | 20 CSVs, 41,855 rows. |
| **5. Env** | `cp .env.example backend/.env` | Every variable is documented in `.env.example`, and the defaults match docker-compose. Add `GEMINI_API_KEY` for live AI (optional). |
| **6. Install** | `cd backend && npm install` | Backend dependencies. |
| **7. Migrate** | `npm run migrate` | Our additive migrations `001`–`004`. Idempotent. |
| **8. Build UI** | `npm run build:web` | Builds `frontend/` into `frontend/dist`. |
| **9. Start** | `npm start` | API + web app on one port. |
| **10. Open** | **http://localhost:3000** | Health: `/api/health`. |

Steps 3–4 need `DATABASE_URL` in the shell:

```bash
export DATABASE_URL=postgresql://postgres:postgres@localhost:5433/kognivera          # bash
$env:DATABASE_URL="postgresql://postgres:postgres@localhost:5433/kognivera"          # PowerShell
```

| Note | |
|------|---|
| **No Gemini key** | Everything else works. The five demo queries are pre-cached (paste one exactly), and other English queries use the heuristic parser. |
| **UI development** | `cd frontend && npm run dev` serves on http://localhost:5173 and proxies `/api` to :3000. |
| **Reset data** | `python data-model/tools/load_data.py --csv-dir data-model/seed/csv --truncate`, then `npm run migrate`. |
| **Dates** | Seed inventory covers **2026-09-01 → 2026-11-29**, so search inside that window. "Goa" is stored as **Panaji**. |
| **Multi-process** | `npm run start:cluster` (`WEB_CONCURRENCY` × `PG_POOL_MAX` must stay under 100). |

---

## 7. Demo Path

The terminal outcome is a confirmed booking that survives concurrency, retries and partial failure, with the database showing
**zero oversell**. A timed script with fallbacks is in [`docs/DEMO_SCRIPT.md`](docs/DEMO_SCRIPT.md).

| # | Screen | Click-path | What it proves |
|---|--------|-----------|----------------|
| 1 | **Login** | Pick any of the 10 seeded travellers → **Sign in** | Identity for the session (no password). |
| 2 | **Home** | **AI search** (the pill at the top-left of the Where box) → paste `जयपुर में 2 रातों के लिए होटल, ₹5000 से कम` | Hindi natural language → real rooms with live availability. The parser badge shows who answered. |
| 3 | **Stay** | **View stay** on a hotel marked "Only N left" → pick room + rate plan → **Add to trip** | A draft: nothing is held and the inventory count is unchanged. |
| 4 | **My trip** | **Add a flight** → **Add to trip** → **Reserve for 10 min** | One atomic hold for hotel + flight with one shared countdown. **Pay now** unlocks only now. |
| 5 | **My trip** | **Demo controls** → *Simulate a failure at checkout* → *Flight sells out (hotel is rolled back)* → **Fill test details** → **Pay now** | **Saga compensation.** The hotel line is compensated and restocked, so nothing is left half-booked. |
| 6 | **My trip** | Reserve again → **Fill test details** → **Pay now** | A confirmed booking with a reference (the terminal outcome). |
| 7 | **Confirmation** | **Retry the same request** | **Idempotency:** "Same booking returned. Nothing was booked twice." |
| 8 | **My bookings** | **Cancel booking** → switch currency in the header | **Restock** + refund, then localised prices. |
| 9 | **Load test** | The scarcest room is pre-selected → **Fire N requests** | **Zero oversell.** The verdict reads the database: *"Zero oversell: all checks passed"*. |
| 10 | **System Visualizer** | **Live backend** → race / saga / retry | The same guarantees, animated from real server responses. |

**Optional scenarios**

- **Two users, one room.** Tab A and tab B sign in as different travellers and add the same last-unit room. A clicks **Reserve** and succeeds. B clicks **Reserve** and is rejected with a sold-out message, and nothing is held. A third tab signs in as **Operations dashboard** to see live holds, who was rejected and **0 violations**. **Reset demo** restores stock.
- **One-stop flights.** Flights → From Bengaluru, To Jaipur, 5 Oct → **One-stop options** (via New Delhi). Both legs are held in one atomic request, so a sold-out second leg leaves nothing held.

---

## 8. Tests / Proof

### Automated tests (60, against real Postgres)

```bash
cd backend
npm test            # 60 tests in ../tests/, ~16 s
                    # all pass with GEMINI_API_KEY blank; with a live key, 4 offline-fallback AI tests skip
npm run invariants  # oversold / negative / held_drift / booked_drift — all must be 0
```

| File | Tests | What it proves |
|------|------:|----------------|
| `holds.test.js` | 14 | **200 concurrent requests for the last 3 units → exactly 3 succeed, 197 sold out.** 25 simultaneous retries of one key → one hold. Atomic multi-night and trip holds. No deadlocks. TTL expiry. |
| `bookings.test.js` | 15 | 20 simultaneous confirm retries → one booking. All three saga failure modes compensate. 12 simultaneous cancels restock once. FX and rate plans. 60 hold-then-book → exactly 3. |
| `api.test.js` | 9 | The HTTP contract, Hindi errors, field-level 400s. **300 concurrent HTTP requests → exactly 3**, and sending every request 3× with the same key never double-books. |
| `session.test.js` | 6 | Personas, isolation between users, **two users racing for the last unit** (one 201, one 409), operator-only endpoints, reset. |
| `connections.test.js` | 4 | Layover rules. Both legs held together, and a sold-out leg leaves nothing held. |
| `ai.test.js` | 8 | Heuristic parsing, **grounded results**, asking instead of guessing, flight roles. |
| `errors.test.js` | 4 | Pool timeout or database down → `503 contention_timeout`, never a 500. |

### The hard-proof test: *"a load test that shows zero oversell"*

```bash
# with the server running (npm start in another terminal):
npm run loadtest -- --requests 500     # 500 concurrent holds for the last units → exactly the free units granted
```

Each run checks six things against the **database**, never the responses:

| Check | Meaning |
|-------|---------|
| `no_oversell` | `booked + held ≤ total` on every row in the database. |
| `exactly_free_units_granted` | Exactly the free units were granted: no more, and no fewer. |
| `no_request_granted_twice` | No request received two holds (also run with every request sent 3× on the same key). |
| `counters_reconcile` | `held_units` equals the sum of active holds, checked against the `holds` table. |
| `db_check_never_fired` | The application enforced the rule itself and the database CHECK never had to reject a write. Removing `FOR UPDATE` makes this fail. |
| `no_transport_or_server_errors` | Every request came back as a success or a clean sold-out. |

Measured on the dev machine: **3 granted · 497 sold_out · 0 errors, all PASS**. The result is the same at 1,000 requests.

### More proofs

| Proof | Run |
|-------|-----|
| In-app race with a live chart and verdict | **Load test** page · `POST /api/loadtests` |
| Mixed confirm / abandoned-hold closing balance (fails if the expiry sweep is off) | `npm run loadtest:mixed` |
| k6, with `teardown()` checking `/api/invariants` | `k6 run scripts/k6-loadtest.js` (from `backend/`) |
| Idempotency after a dropped response | `node scripts/simulate-network-retry.mjs --runs 8 --abort-ms 5` |
| From **separate machines** (GitHub Actions; needs a public URL) | `gh workflow run distributed-load-test.yml` · `distributed-idempotency-test.yml` · `network-retry-test.yml` |
| Schema conformance | `python data-model/tools/validate_postgres.py --csv-dir data-model/seed/csv --conformance-script data-model/tools/validate_conformance.py` |
| Saved k6 report | [`docs/evidence/k6-report.html`](docs/evidence/k6-report.html) |

Test index: [`tests/README.md`](tests/README.md).

---

## Repository Layout

```
README.md            this file                      .env.example        every variable, dummy values
frontend/            React UI                       data-model/         schema, migrations, seed, DATA_MODEL.md
backend/             API, workers, load-test tools  ai/                 NL search: code + prompts
tests/               automated tests                docs/               architecture, design, extras, demo script
.github/workflows/   distributed proofs             docker-compose.yml  local Postgres
```
