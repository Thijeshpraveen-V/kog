# Kognivera — Distributed Booking & Inventory (APS-05)

## 1. Team & Problem Statement

- **Team:** `--rebase` — Thijesh, Tejas, Yogita, Neha
- **Problem statement:** **APS-05 — Distributed Booking & Inventory System.** A booking and inventory service that
  never oversells under concurrency. Full text: [`docs/problem-statement/ps_description.md`](docs/problem-statement/ps_description.md).

## 2. What we built

Mapped to the PS's "What you need to build" checklist — each item is wired end to end and reachable from the app:

- **Availability search with a TTL hold** — hotel + flight search (form and natural language); items go into **My trip** (nothing held yet), then one **Reserve** holds the whole trip atomically — hotel and flight share a single server-clamped TTL (5 s – 30 min) and one live countdown.
- **Confirm with payment; release on timeout** — `POST /api/bookings` captures a (mock) payment; a background sweep expires abandoned holds, and a late confirm is refused at confirm time regardless of the sweep.
- **No overbooking under concurrent requests** — one transaction: `SELECT … FOR UPDATE` on the inventory row(s), check, then update (on the hot path a single Postgres function). The DB `CHECK` is only a safety net and is asserted never to fire.
- **Idempotent booking API** — `Idempotency-Key` on every write, `INSERT … ON CONFLICT DO NOTHING`; a retried request returns the original result and never books twice (concurrent retries and retry-after-dropped-response are both tested).
- **Multi-item saga with compensation** — hotel + flight in one booking; if any line or the payment fails, every confirmed line is rolled back and the response says why.
- **Cancellation with restock; localised currency** — cancel refunds and returns units to the pool; prices convert through the dated `fx_rates` table (INR / USD / EUR / …), original amounts never overwritten.
- **The standout deliverable — a load test that shows zero oversell** — in-app (Load test page and the live System Visualizer), k6, GitHub Actions from separate machines, and a mixed confirm / abandoned-hold scenario that ends with a closing-balance assertion.

## 3. Architecture

```
frontend/ (React + Vite, en/हिन्दी) ──/api──► backend/ (Express 5) ──pg pool──► PostgreSQL 16
                                                 │  routes → validation → booking · inventory · payment · loadtest
                                                 │  workers/expiry (30 s sweep, advisory-locked)
                                                 └─imports─► ai/ (LangChain tool-calling agent, OpenRouter/gpt-4o-mini,
                                                                prompts + heuristic fallback)
```

One process serves the API and the built web app. Component-by-component notes, how each guarantee is met, the API
table and the known limitations: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md). Deep-dive with measured numbers:
[`backend/README.md`](backend/README.md).

## 4. Data model

We use the **canonical APS-05 model unchanged** (names and columns kept). Tables the code touches:
`inventory_calendar` (the contended resource), `holds`, `bookings`, `booking_items`, `payments`, `hotels`,
`hotel_room_types`, `hotel_rate_plans`, `cities`, `flights`, `flight_fares`, `airports`, `airlines`, `users`,
`currencies`, `fx_rates`.

**Tables we added (all additive, rule R1):** `load_test_runs`, `load_test_results`, `search_logs`.
**Columns / constraints / indexes we added:** `booking_items.hold_id`, `load_test_runs.snapshot`,
`CHECK (booked_units >= 0 AND held_units >= 0)`, a partial index on active holds, and the
`kognivera_create_holds()` function. Nothing canonical is renamed, dropped or repurposed.

Full mapping, why each addition exists, and where every boundary rule is enforced in code and tested:
[`data-model/DATA_MODEL.md`](data-model/DATA_MODEL.md).

## 5. AI features

| Capability | Mechanism | How it is grounded |
|---|---|---|
| **Natural-language search** in English and Hindi, for hotels (e.g. "3-star hotel in Jaipur for 2 adults, Oct 10-12, under ₹5000") **and flights** (e.g. "Bengaluru to Jaipur on Nov 4, 2 seats") | A **LangChain tool-calling agent** (`createReactAgent`) against OpenRouter's OpenAI-compatible endpoint (`openai/gpt-4o-mini` by default), with three read-only tools — `resolve_city`, `search_hotels`, `search_flights` (`ai/src/tools.js`, prompt `ai/prompts/agent_system.md`); the flight search endpoint (`kind: "flights"`) uses the same agent restricted to `resolve_city` + `search_flights` only | The agent has **no tool that can hold, book or invent inventory** — every result comes from a tool call that ran the same grounded SQL the form uses. A structured `{language, confidence, reasoning_summary}` follow-up call reports how sure the agent was, without ever hiding a real result behind it. City/route names are constrained to what's really in the database. |
| **"You might also consider"** — nearby dates, an earlier checkout, or a nearby hotel | Deterministic, non-LLM (`backend/src/modules/inventory/alternatives.js`) | When the top hotel result is scarce (≤2 rooms): real nearby check-in dates (±3 days) with more availability, the real date active holds/bookings on it next check out (`checkout_date`), and up to 3 real nearby hotels in the same city (haversine on `hotels.lat/lng`) that are actually bookable for the same stay. When a hotel search is empty: a real check of which single filter (price/stars/breakfast/refundable) is the binding constraint. Applies to both AI and plain structured search, since it lives inside `searchHotels()` itself. |
| **Multi-hop flight connections** | Deterministic, non-LLM (`searchFlightConnections` in `backend/src/modules/inventory/search.js`) | Most city pairs in this dataset have no direct flight at all (~1,070 of ~6,300 possible ordered pairs do). Every flight search — plain (`GET /search/flights`) and AI — always returns real one-stop itineraries (origin→X→destination, 45min–12h layover, both legs genuinely bookable) as `connections` alongside any direct results, never only as a fallback when direct is scarce; each leg is addable to the trip exactly like a direct flight. |
| **Comparison summary** of the top results | LLM text generation via the same OpenRouter client (`ai/prompts/summarise.md`) | Prompt says "use only this data" and receives only the top 3 result rows; failure never fails the search. |
| **Graceful degradation** | Pre-seeded cache for the demo queries → agent → small English heuristic parser | Every response reports `parser: cache \| agent \| heuristic`, and the UI shows which one answered. |

Code: [`ai/search.js`](ai/search.js). Tests: `tests/ai.test.js`.

## 6. Run it locally

Prerequisites: Node ≥ 20, Docker Desktop, Python 3 (for the seed loader).

```bash
# 1. Postgres (port 5433)
docker compose up -d

# 2. Schema + seed data (canonical schema, then the 20 CSVs = 41,855 rows)
pip install -r data-model/tools/requirements.txt
export DATABASE_URL=postgresql://postgres:postgres@localhost:5433/kognivera
python data-model/tools/apply_schema.py --schema data-model/schema.sql
python data-model/tools/load_data.py --csv-dir data-model/seed/csv

# 3. Backend: env, our additive migrations, web build, start
cp .env.example backend/.env              # add GEMINI_API_KEY for live AI search (optional — see below)
cd backend
npm install
npm run migrate                           # applies data-model/migrations/*.sql, idempotent
npm run build:web                         # builds frontend/ into frontend/dist
npm start                                 # → http://localhost:3000
```

Open **http://localhost:3000**. Without `OPENROUTER_API_KEY` the app still runs: the five demo queries in `ai/search.js` (`seedDemoQueries`) are pre-cached — paste one exactly
and other English queries use the heuristic parser (Hindi free text needs the key).
For live front-end development run `npm run dev` in `frontend/` (port 5173, proxies `/api` to :3000).

## 7. Demo path

The exact click-path (≈6 min) that reaches the terminal outcome — a booking that survives concurrency, retries and
partial failure. A timed run-through with speaker notes and fallbacks: [`docs/DEMO_SCRIPT.md`](docs/DEMO_SCRIPT.md).

1. **Search** — on Home click **AI search** (top-left of the Where box), paste `जयपुर में 2 रातों के लिए होटल, ₹5000 से कम` → results are real rooms with live availability ("AI" / "Cached" badge).
2. **Add to trip** — **View stay** on a hotel marked "Only N left" → pick a room and rate plan → **Add to trip**. Nothing is held yet (the inventory count is unchanged).
3. **Reserve + saga rollback** — **Add a flight** (Add to trip), then **Reserve for 10 min**: one atomic hold, one shared countdown, and only now does **Pay now** unlock. Open **Demo controls** → *Simulate a failure at checkout* → *Flight sells out (hotel is rolled back)* → **Fill test details** → **Pay now** → the hotel is compensated and its room returns to inventory; nothing is left half-booked.
4. **Idempotency** — on a confirmed booking click **Retry the same request** → "Same booking returned. Nothing was booked twice."
5. **Cancel + restock** — **My bookings → Cancel booking** → refunded, room back on sale; switch currency in the header to see localised prices.
6. **Zero oversell (the standout proof)** — **Load test** page → the scarcest room is pre-selected → **Fire N requests** → the verdict reads the database, not the responses: *"Zero oversell: all checks passed"*. To watch the same guarantees animate from real data, open **System Visualizer → Live backend** and run the race, saga or duplicate-retry scenario.

7. **Two users, one scarce room (mock login)** — the app opens on a **demo login** (no password): pick one of 10 real seeded travellers. Each browser tab keeps its own session. Tab 1: sign in as user A; tab 2: sign in as user B. Both add the same last-unit room to their trip (search hides a room once it is fully held, so add it first), then A clicks **Reserve** (succeeds) and B clicks **Reserve** (rejected: sold out, room named, nothing held). Third tab: sign in as **Operations dashboard** → live holds, bookings, inventory counts, the room in play, who was rejected, and **0 violations**. **Reset demo** returns the stock afterwards.

## 8. Tests / proof

```bash
cd backend
npm test                     # 70 tests, real Postgres (66 pass, 4 skip when a live OpenRouter key is set)
npm run invariants           # data invariants: oversold / negative / held_drift / booked_drift — all must be 0

# the next two need the server running (`npm start` in another terminal):
npm run loadtest             # the hard proof, CLI: N concurrent holds for the last units → exactly the free units granted
npm run loadtest:mixed       # mixed scenario: winners confirm, the rest abandon and expire → closing balance PASS/FAIL
```

- **The hard-proof test** (PS: *"a load test that shows zero oversell"*): `tests/holds.test.js` (200 concurrent requests for the last 3 units → exactly 3 succeed, 197 sold out), `tests/api.test.js` (300 over HTTP; every request sent 3× with the same key never double-books), and the in-app Load test tab.
- **Mixed scenario** ([`backend/scripts/mixed-loadtest.mjs`](backend/scripts/mixed-loadtest.mjs)): after a race, half of the winners confirm and half abandon their holds past the TTL; it then asserts `booked + held + available == total` against figures derived from the individual hold records, that no expired hold still holds stock, and zero oversell throughout. It also *fails* when the expiry sweep is disabled — the assertion is not vacuous.
- **From separate machines:** `.github/workflows/distributed-load-test.yml` and `distributed-idempotency-test.yml` fire from a GitHub Actions matrix, then verify against the database (`gh workflow run …`).
- **k6:** `k6 run scripts/k6-loadtest.js` (from `backend/`) — its `teardown()` checks `/api/invariants` directly.
- **Schema conformance:** `python data-model/tools/validate_postgres.py --csv-dir data-model/seed/csv --conformance-script data-model/tools/validate_conformance.py` (row counts are exact against a freshly seeded database; load-test and demo rows are extras by design, since nothing is ever hard-deleted).
- Test index and what each file covers: [`tests/README.md`](tests/README.md).

## Repository layout

```
README.md            this file                      .env.example        every variable, dummy values
frontend/            React UI                       data-model/         schema, migrations, seed, DATA_MODEL.md
backend/             API, workers, load-test tools  ai/                 NL search: code + prompts
tests/               automated tests                docs/               architecture, demo script, design, PS
.github/workflows/   distributed proofs             docker-compose.yml  local Postgres
```
