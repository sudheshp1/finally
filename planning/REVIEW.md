# Review of `planning/PLAN.md`

Reviewed against the current working-tree version of PLAN.md and the built market data code in `backend/app/market/`. The plan is in good shape: it has clear contracts, sensible scope, and error codes and response shapes are mostly pinned down. The items below are ordered by how likely they are to cause rework or bugs if left as-is.

---

## 1. Things that will break or contradict each other

### 1.1 Buying an untracked ticker cannot work as written (§7, §6)
§7 says "a buy that opens a new position adds the ticker to the market data source if it isn't already tracked", but the same section says a trade with no cached price fails with `PRICE_UNAVAILABLE`. An untracked ticker never has a cached price, so the auto-track rule can never fire, and the trade bar can only buy tickers already on the watchlist (or already held).

**Decide one:**
- (a) **Simplest:** buys require a tracked ticker; drop the auto-track sentence. The trade bar error says "Add TICKER to your watchlist first" (or the UI only lets you trade tracked tickers). The chat flow already handles this by ordering watchlist changes before trades.
- (b) The buy path calls `source.add_ticker(t)` *before* reading the price. That works with the simulator (it seeds the cache right away) but still fails with Massive until the next poll, and it leaves a tracked ticker behind if the buy then fails.

### 1.2 Docker volume: named volume or bind mount? (§11, §4)
§11 runs `-v finally-data:/app/db`, which is a **named volume**. The next sentence says "the `db/` directory in the project root maps to `/app/db`", which is a **bind mount**. §4 also describes top-level `db/` as "the runtime volume mount point" with a `.gitkeep`. Pick one:
- Named volume: simpler and avoids Windows path-quoting and permission problems in `start.ps1`. Drop `db/.gitkeep` and the "maps to project root" sentence.
- Bind mount: the user can see and delete `finally.db`, but scripts must pass an absolute path (`$(pwd)/db`, `${PWD}\db`).

### 1.3 The DB path is ambiguous, and `backend/db/` vs `db/` will collide (§4, §7)
- The backend's runtime path to `finally.db` isn't stated. If it's relative (`db/finally.db`) and someone runs `uv run uvicorn ...` from `backend/`, the file lands in `backend/db/`, the directory that holds schema and seed code.
- **Suggest:** a `DB_PATH` env var (default `/app/db/finally.db` in the container, `../db/finally.db` locally), and rename the code package to something like `backend/app/database/` so "db" means only the runtime directory.

### 1.4 Lifespan ordering vs router registration (§3 step 5)
`create_stream_router(price_cache)` needs the cache, but step 3 creates the cache inside the lifespan, and step 5 "mounts API routers" there too. Routers are normally included at app construction, not during startup. **Suggest:** create the `PriceCache` (and market source object) at app-construction time or put them on `app.state`. Include all routers plus the static mount when the app is built. The lifespan should only *start and stop* things: DB init, initial snapshot, `source.start(tickers)`, snapshot task.

### 1.5 Prices "stored as received" vs the built code (§3 Conventions)
`PriceCache.update()` rounds `price` and `previous_price` to 2 dp. Either update the convention ("prices are rounded to 2 dp in the cache") or change the code. Rounding in the cache is fine and simpler.

### 1.6 Lightweight Charts rejects duplicate or unsorted timestamps (§10)
- **P&L chart:** a post-trade snapshot and a 30 s snapshot can land in the same second, and several trades can happen within one second. Lightweight Charts requires strictly increasing `time` values (whole Unix seconds by default) and throws otherwise. The frontend must convert ISO to seconds and de-duplicate (keep the last value per second). Alternatively, the backend can guarantee uniqueness.
- **Main chart from SSE:** ticks arrive every ~500 ms with float timestamps, so two points per second. The same de-duplication (or 1-second bucketing) is needed. Worth one sentence in §10 so the Frontend agent doesn't hit this at runtime.

### 1.7 `.gitignore` will silently hide frontend code
The root `.gitignore` is the stock Python template. It contains `lib/`, `build/`, `dist/` and `downloads/`, which will ignore `frontend/src/lib/` (a very common Next.js folder) without warning. It also does **not** ignore `node_modules/`, `.next/`, `out/`, `db/*.db` or `finally.db` (it only covers `db.sqlite3`). Add a task to fix `.gitignore` before frontend work starts: anchor the Python patterns to `/backend/` or remove `lib/`, and add the Node and SQLite entries.

### 1.8 `--env-file .env` fails if `.env` doesn't exist (§5, §11)
`docker run --env-file .env` errors out when the file is missing, yet §5 says the app works without an OpenRouter key. The start scripts should create `.env` from `.env.example` when it's missing, or omit `--env-file`. Also: `.env.example` doesn't exist in the repo yet; it should be listed as a deliverable. Pick **one** mechanism (`--env-file`). §5's "mounted into the container *or* `--env-file`" leaves it open, and the backend shouldn't also try to read a `.env` file inside the container.

### 1.9 Node 20 is end-of-life (§11)
Node 20 reached EOL in April 2026. Use Node 22 or 24 LTS (`node:22-slim`), and pin major versions of Next.js, Tailwind and Lightweight Charts in the plan. Tailwind v3 vs v4 changes how the "custom dark theme" is configured, and Lightweight Charts v4 vs v5 changes the series API (`addLineSeries` vs `addSeries(LineSeries)`).

---

## 2. Gaps and open questions

### API and error handling
1. **FastAPI's default 422 responses** don't match the §8 error shape. The plan needs a `RequestValidationError` handler (map it to `400` with a code) and a catch-all handler for `500` (e.g. `INTERNAL_ERROR`). Missing codes: invalid `side` (e.g. `INVALID_SIDE`, or reuse a generic `VALIDATION_ERROR`), malformed JSON body, empty chat message.
2. **Quantity rounding edge case:** is `0.00001` rejected (`INVALID_QUANTITY`) after rounding to 4 dp? Suggest: round first, then validate `> 0`.
3. **Close-position threshold:** quantities are rounded to 4 dp, yet positions close below `1e-9`. With 4 dp rounding applied to the stored quantity, the check can simply be `quantity == 0` after rounding (or `< 1e-4`). Also specify rounding for `avg_cost` (suggest 4 dp or none) and for sell proceeds (2 dp, same as costs).
4. **Concurrency:** a manual trade and a chat trade can run at once. As long as the cash check and the write happen in one synchronous block with no `await` between them, asyncio makes this safe. Say so explicitly, or require an `asyncio.Lock` around trade execution. Otherwise an agent may write `await` between the read and the write and double-spend cash.
5. **`DELETE /api/watchlist/{ticker}`:** is the path param normalized to uppercase and regex-validated (`INVALID_TICKER`)? Presumably yes; state it.
6. **`POST /api/watchlist` response:** is it the same shape as a `GET /api/watchlist` item (including `price` and `session_change_percent`)? With the simulator the price is available immediately; with Massive it's `null`.
7. **Unknown symbols:** any regex-valid string (e.g. `ZZZZ`, or even `.`) can be added. The simulator invents a price; Massive leaves it `null` forever. Is that acceptable? Tightening the regex to `^[A-Z][A-Z.]{0,5}$` at least rejects `.` and `...`.

### Chat
8. **Persistence on failure:** if the LLM call fails (`LLM_ERROR`) or is unavailable (`LLM_UNAVAILABLE`), is the user message still saved to `chat_messages`? Suggest **not** saving anything on failure, so that history never contains an unanswered message, and having the frontend show the error inline.
9. **No-op actions:** what's the status of "add PYPL" when PYPL is already on the watchlist, or "remove X" when X isn't on it? Suggest: an add of an existing ticker is `executed` (idempotent, matching REST), and a remove of a missing ticker is `failed` with `NOT_FOUND`. Watchlist change results should carry `error`/`code` on failure, the same way trades do; the example only shows the success shape.
10. **LLM timeout:** give a concrete value (e.g. 30 s) and say what maps to `LLM_ERROR`.
11. **Blocking call:** the cerebras skill's example uses the synchronous `litellm.completion`. Called directly inside an `async def` endpoint, it blocks the event loop, which freezes SSE price streaming for every client while the LLM thinks. The plan should require `litellm.acompletion` or `asyncio.to_thread(...)`.
12. **Schema strictness:** strict JSON-schema structured outputs generally require every property to be listed as required. Make `trades` and `watchlist_changes` **required arrays that may be empty**, not optional fields. That makes the schema strict-mode-compatible and simplifies parsing.
13. **Mock-rule matching:** specify the regex. Does the quantity allow decimals? Is there a word boundary, so "address" doesn't match `add` and "I'd like to buy 1 aapl" works? Suggest `\bbuy\s+(\d+(?:\.\d+)?)\s+([A-Za-z.]+)\b`, and so on. How are `LLM_MOCK` values parsed (`"true"`, `"TRUE"`, `"1"`)?
14. §9 says "There is an OPENROUTER_API_KEY in the .env file", which reads as a requirement and conflicts with §5, where the key is optional. Reword.

### Market data
15. **Massive's "session open" is not the market open.** With Massive, `session_open` is the first price polled since container start, so the watchlist "change %" column means "since I started the app", not "today". That's acceptable for a demo, but it should be a conscious choice. The Massive snapshot response includes a day-change field, which could be used instead (at the cost of the two sources diverging).
16. **SSE shutdown:** infinite `StreamingResponse` generators can stall uvicorn's graceful shutdown until Docker sends SIGKILL after 10 s. Consider `--timeout-graceful-shutdown 3` in the CMD. This is minor, but it makes `stop.sh` feel snappier.

### Frontend
17. **The SSE payload includes non-watchlist tickers** (held positions removed from the watchlist). The frontend must render the watchlist from `/api/watchlist`, not from SSE keys. Add to Data Flow: "the watchlist is fetched on load and refetched after any watchlist change (manual or chat)". Today the Data Flow section only covers prices and portfolio.
18. **The connection status "red" state rarely happens.** `EventSource` keeps retrying forever (`CONNECTING`) after network errors. It only reaches `CLOSED` on a non-200 response or a wrong content type. If the server is down, the indicator stays yellow. Consider "red if reconnecting for more than N seconds", or accept yellow as the realistic disconnected state.
19. **Unbounded in-memory series:** cap the sparkline and main-chart buffers (e.g. last 300–1000 points per ticker). Otherwise a tab left open overnight grows without bound.
20. **Positions table "% change":** this presumably means `pnl_percent` (vs avg cost), not the session change. Say so.
21. **Initial selected ticker** for the main chart: the first watchlist ticker?
22. **Main chart history:** it only shows data since page load, with no backfill endpoint. That's fine; confirm it's intended so nobody builds a history endpoint.
23. **Units:** state once that all `*_percent` fields are percentages (0.27 = 0.27%), not fractions. Also give the rounding: the SSE example has 4 dp (`0.2737`), the watchlist example has 2 dp (`0.27`).

### Database
24. **Seeding rule:** "seed if the file doesn't exist or is empty" must not reseed the default watchlist when the user has deliberately removed every ticker. Suggest: seed only when the `users_profile` row for `"default"` is created (`INSERT ... ON CONFLICT DO NOTHING` for the user, and watchlist rows only in that same first-run branch).
25. **Snapshot growth:** a row every 30 s is ~2,900 rows/day, forever, while history returns only the last 500 (~4 hours). Either prune old rows (e.g. keep 7 days) or accept the growth. Note that the P&L chart therefore spans at most ~4 hours.

### Docker, scripts and dev workflow
26. **Local development workflow is undefined.** Running `next dev` (port 3000) against FastAPI (port 8000) means cross-origin `/api` calls. Next.js `rewrites` don't apply under `output: 'export'` and also buffer SSE. Pick one: (a) always build the frontend and let FastAPI serve it, or (b) add an `NEXT_PUBLIC_API_BASE` setting plus dev-only CORS. Also, the `StaticFiles(directory="static")` mount raises at startup if `static/` doesn't exist. Mount it only if the directory exists, so the backend can run and be tested without a frontend build.
27. **Security:** there's no auth, and the container holds a paid API key. Have the start scripts publish `-p 127.0.0.1:8000:8000` so other machines on the LAN can't spend the user's OpenRouter credits.
28. **Start-script idempotency:** name the container (`--name finally`), remove any existing container with that name before `run`, and decide whether "build if not built" can leave a stale image after code changes. Suggest always running `docker build`, which is fast thanks to the layer cache.
29. **Health check:** the slim Python image has no `curl`. If a Docker `HEALTHCHECK` is wanted, use `python -c "import urllib.request; ..."`. Also `uv sync --frozen --no-dev`.
30. **E2E isolation:** `docker-compose.test.yml` should use a fresh volume or `tmpfs` for `/app/db`, so the "fresh start" scenario is deterministic and test runs don't leak state. It should also wait on `/api/health` before Playwright starts.

### Process
31. **No build order or agent roster.** The plan mentions "Frontend Engineer" and "Backend/Market Data" agents but has no list of agents, phases or a definition of done. A short milestone list would help agents coordinate through `planning/`, for example: (1) the two market-data follow-ups and `.gitignore` fixes; (2) DB + portfolio + watchlist API with tests; (3) chat/LLM with mock mode; (4) frontend; (5) Docker and scripts; (6) E2E.
32. **Shared contract:** consider having the backend expose its OpenAPI schema (FastAPI does this for free) and having the frontend generate or hand-write TS types from §8. That keeps the response shapes from drifting.

---

## 3. Opportunities to simplify

1. **Drop auto-tracking on buy** (see 1.1). Buys require a tracked ticker. This removes a code path and a test case, and the tracked-ticker rules become: tracked = watchlist ∪ positions, re-evaluated on watchlist removal and on position close.
2. **Drop the structured-output fallback chain** in §9 (JSON-mode fallback, retry, raw-text fallback). The cerebras skill already passes a Pydantic model as `response_format` for this exact model and provider. Keep one rule: "if validation fails, return the raw text as `message` with no actions". The retry and JSON-mode branch are speculative complexity.
3. **Make `POST /api/watchlist` always return `200`** (or always `201`) with the item, instead of 201 for new and 200 for existing. The frontend doesn't care, and it's one less branch and test.
4. **`GET /api/watchlist` could return only `{ticker, added_at}`.** SSE delivers prices within 500 ms, so the frontend already has to merge SSE data in. If the price fields stay (for first paint), that's fine, but they're redundant.
5. **Rename `users_profile` to `users`** (and keep `id`), or keep the name but stop explaining the `id`/`user_id` asymmetry. It's minor, but one fewer thing to get wrong in joins.
6. **Move `rich` to the dev dependencies** in `backend/pyproject.toml`. It's only used by `market_data_demo.py` and shouldn't ship in the production image.
7. **Choose a named volume** (see 1.2). It removes `db/.gitkeep`, the gitignore rule for `finally.db`, and the cross-platform path handling in `start.ps1`.
8. **Collapse the close-position threshold** into the 4 dp rounding rule (see 2.3).

---

## 4. Verified as consistent

- The numeric examples in §8 add up: 10 × 191.24 = 1,912.40; cash 10,000 − 1,905.20 = 8,094.80; P&L 7.20; 7.20 / 1,905.20 = 0.378%. In §6, 0.11 / 190.41 = 0.0578% and 0.52 / 190.00 = 0.2737%.
- Follow-up 2 in §6 is accurate: `stream.py` currently declares `router` at module level (line 17) and decorates it inside the factory, so calling the factory twice registers `/prices` twice.
- Follow-up 1 fits the existing design: `remove_ticker()` already calls `cache.remove()` in both sources, so storing `session_open` alongside each cache entry makes it reset naturally when a ticker is removed and re-added, as §6 specifies.
- Massive polls all tracked tickers in one `get_snapshot_all` call, so the free-tier budget (5 calls/min at a 15 s interval) holds no matter how big the watchlist gets.
