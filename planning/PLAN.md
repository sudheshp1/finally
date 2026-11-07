# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist (showing any prior conversation)

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving        │
│                      (Next.js export)           │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background tasks: market data sim/poller,      │
│                    portfolio snapshots          │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

### Startup Sequence (FastAPI lifespan)

Order matters, because the market data source needs to know which tickers to track:

1. Initialize the database (create tables and seed if missing) — see §7
2. Record an initial portfolio snapshot if none exist (so the P&L chart is never empty)
3. Create the `PriceCache` and market data source; start it with the **tracked tickers** (watchlist ∪ open positions — see §6)
4. Start the 30-second portfolio snapshot background task
5. Mount API routers, then mount the static frontend last (`StaticFiles(directory="static", html=True)` at `/`)

On shutdown: stop the snapshot task and the market data source.

### Conventions

- **Timestamps**: ISO 8601 UTC strings in the database and all REST responses. SSE events use Unix seconds (float) for compactness.
- **Money**: cash and trade costs are rounded to 2 decimal places when stored. Prices are stored as received.
- **Quantities**: must be > 0; rounded to 4 decimal places. Fractional shares are supported.
- **Tickers**: normalized to uppercase; must match `^[A-Z.]{1,6}$`.
- **Errors**: every non-2xx API response uses one shape (see §8).

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, init logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start.sh              # Launch Docker container (macOS/Linux)
│   ├── stop.sh               # Stop Docker container (macOS/Linux)
│   ├── start.ps1             # Launch Docker container (Windows PowerShell)
│   └── stop.ps1              # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

There is no root `docker-compose.yml`; the start scripts are the one supported way to launch. The only compose file is `test/docker-compose.test.yml`.

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend initializes the database at startup — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# OpenRouter API key for LLM chat functionality
# (the app runs without it; only chat is unavailable)
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests); no API key needed
- If `OPENROUTER_API_KEY` is missing and `LLM_MOCK` is not `true` → the app still starts normally; market data, trading and the watchlist all work, and `POST /api/chat` returns `503` with code `LLM_UNAVAILABLE` and a friendly message the chat panel displays
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

> **Status:** built and tested — see `planning/MARKET_DATA_SUMMARY.md`. Two small follow-ups are listed at the end of this section.

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.); unknown tickers get a random seed price
- New tickers are priced immediately on `add_ticker`
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for all tracked tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- A newly added ticker has no price until the next poll

### Tracked Tickers

The market data source tracks the **union of the watchlist and all open positions**. This keeps held positions priced (for valuation and selling) even after they're removed from the watchlist.

- Adding to the watchlist → `source.add_ticker(t)`
- Removing from the watchlist → `source.remove_ticker(t)` **only if** there is no open position in `t`
- Selling a position to zero → `source.remove_ticker(t)` **only if** `t` is not on the watchlist

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, session open price (the first price seen for the ticker since startup), and timestamp for each ticker
- SSE streams, portfolio valuation and trade execution all read from this cache
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The server checks the cache every ~500ms and, when anything has changed, sends **one event containing all tracked tickers**, keyed by ticker
- First message is `retry: 1000` so the browser reconnects after 1s
- Client handles reconnection automatically (EventSource has built-in retry)

Event payload:

```
data: {"AAPL": {"ticker": "AAPL", "price": 190.52, "previous_price": 190.41,
                "session_open": 190.00, "timestamp": 1762500000.123,
                "change": 0.11, "change_percent": 0.0578,
                "session_change_percent": 0.2737, "direction": "up"},
       "GOOGL": {...}, ...}
```

- `change` / `change_percent` — since the previous tick (use for the flash direction)
- `session_change_percent` — since `session_open` (shown as the "change %" column in the watchlist)
- `direction` — `"up"`, `"down"` or `"flat"`

### Follow-ups to the Built Code

1. Add `session_open` and `session_change_percent` to `PriceUpdate` / `PriceCache` (the cache records the first price per ticker and keeps it until the ticker is removed).
2. In `stream.py`, create the `APIRouter` inside `create_stream_router()` instead of at module level, so calling the factory twice (e.g. in tests) doesn't register the route twice.

---

## 7. Database

### SQLite, Initialized at Startup

During the FastAPI lifespan startup (before the market data source starts — see §3), the backend checks the SQLite database. If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

Every table is scoped to a user: `users_profile` by its `id`, and all other tables by a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

Tables with a natural key use it as the primary key (which makes upserts a simple `INSERT ... ON CONFLICT DO UPDATE`). Append-only tables use UUIDs.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`) — this is the user id
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- PRIMARY KEY `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- PRIMARY KEY `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded once at startup if the table is empty, every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trade and watchlist results, including failures, in the same shape as the `/api/chat` response; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

### Trade Execution Rules

Shared by `POST /api/portfolio/trade` and AI-initiated trades:

- **Price**: the current cached price for the ticker. If there is none (ticker not tracked, or Massive hasn't polled it yet), the trade fails with `PRICE_UNAVAILABLE`.
- **Buy**: requires `cash_balance >= round(quantity × price, 2)`, otherwise `INSUFFICIENT_CASH`. New `avg_cost` is the quantity-weighted average of the old position and the new shares.
- **Sell**: requires held quantity ≥ requested quantity, otherwise `INSUFFICIENT_SHARES`. `avg_cost` is unchanged by sells. Realized P&L is not tracked.
- **Closing a position**: when quantity falls below `1e-9`, the position row is deleted.
- **Atomicity**: cash update, position update and trade log insert happen in one transaction; a snapshot is recorded right after.
- **Tracked tickers**: a buy that opens a new position adds the ticker to the market data source if it isn't already tracked (see §6).

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates (payload in §6) |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | The most recent 500 portfolio value snapshots, oldest first (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |
| GET | `/api/chat/history` | The most recent 50 chat messages, oldest first (restores the chat panel on reload) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment): `{"status": "ok"}` |

### Error Format

Every non-2xx response uses:

```json
{"error": "Insufficient cash: need $1,905.20, have $1,200.00", "code": "INSUFFICIENT_CASH"}
```

| Code | HTTP | When |
|---|---|---|
| `INVALID_TICKER` | 400 | Ticker doesn't match `^[A-Z.]{1,6}$` |
| `INVALID_QUANTITY` | 400 | Quantity ≤ 0 or not a number |
| `PRICE_UNAVAILABLE` | 400 | No cached price for the ticker yet |
| `INSUFFICIENT_CASH` | 400 | Buy costs more than available cash |
| `INSUFFICIENT_SHARES` | 400 | Sell exceeds held quantity |
| `NOT_FOUND` | 404 | Removing a ticker not on the watchlist |
| `LLM_UNAVAILABLE` | 503 | No API key configured |
| `LLM_ERROR` | 502 | LLM call failed or timed out |

### Response Shapes

**`GET /api/portfolio`**

```json
{
  "cash_balance": 8094.80,
  "positions_value": 1912.40,
  "total_value": 10007.20,
  "unrealized_pnl": 7.20,
  "positions": [
    {"ticker": "AAPL", "quantity": 10, "avg_cost": 190.52, "current_price": 191.24,
     "market_value": 1912.40, "unrealized_pnl": 7.20, "pnl_percent": 0.378}
  ]
}
```

If a position has no cached price, `current_price` falls back to `avg_cost`.

**`POST /api/portfolio/trade`** — request `{"ticker": "AAPL", "quantity": 10, "side": "buy"}`; response `200`:

```json
{
  "trade": {"id": "…", "ticker": "AAPL", "side": "buy", "quantity": 10,
            "price": 190.52, "executed_at": "2026-11-07T14:03:12Z"},
  "cash_balance": 8094.80
}
```

The frontend refetches `/api/portfolio` after a successful trade.

**`GET /api/portfolio/history`**

```json
[{"total_value": 10000.00, "recorded_at": "2026-11-07T14:00:00Z"}, ...]
```

**`GET /api/watchlist`**

```json
[{"ticker": "AAPL", "price": 190.52, "session_change_percent": 0.27, "added_at": "2026-11-07T14:00:00Z"}, ...]
```

`price` and `session_change_percent` are `null` if no price is cached yet.

**`POST /api/watchlist`** — request `{"ticker": "pypl"}` (normalized to `PYPL`); response `201` with the watchlist item. Adding a ticker that's already present returns `200` with the existing item (idempotent).

**`DELETE /api/watchlist/{ticker}`** — response `204`.

**`POST /api/chat`** — request `{"message": "Buy 10 AAPL"}`; response `200`:

```json
{
  "id": "…",
  "role": "assistant",
  "message": "Done — bought 10 AAPL at $190.52.",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10, "status": "executed", "price": 190.52},
    {"ticker": "TSLA", "side": "buy", "quantity": 100, "status": "failed",
     "error": "Insufficient cash", "code": "INSUFFICIENT_CASH"}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add", "status": "executed"}
  ],
  "created_at": "2026-11-07T14:03:12Z"
}
```

**`GET /api/chat/history`** — array of messages. User messages are `{"id", "role": "user", "message", "created_at"}`; assistant messages have the same shape as the `POST /api/chat` response.

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend makes **a single LLM call**:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the most recent 20 messages from the `chat_messages` table. Assistant messages include their `actions` results, so the LLM sees what succeeded or failed in earlier turns.
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Validates the response against a Pydantic model (see below)
6. Auto-executes the actions: **watchlist changes first, then trades** (so "add PYPL and buy 5" works in one response with the simulator)
7. Stores the user message and the assistant message with its action results in `chat_messages`
8. Returns the response shape from §8 to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (§7 Trade Execution Rules)
- `watchlist_changes` (optional): Array of watchlist modifications (`"add"` or `"remove"`)

**Parsing:** if the provider doesn't support `response_format` with a JSON schema for this model, fall back to JSON mode plus Pydantic validation, with one retry on a validation failure. If the output still can't be parsed, return the raw text as `message` with no actions, rather than an error.

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If an action fails validation (e.g., insufficient cash), there is no second LLM call. The failure is returned in that action's result (`status: "failed"`, `error`, `code`) and shown in the chat panel's inline confirmations. It's also stored in `actions`, so the LLM sees it on the next turn.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

Mock rules (case-insensitive, first match wins):

| User message contains | Mock response |
|---|---|
| `buy <qty> <TICKER>` | Message "Buying <qty> <TICKER>." plus one buy trade |
| `sell <qty> <TICKER>` | Message "Selling <qty> <TICKER>." plus one sell trade |
| `add <TICKER>` | Message "Adding <TICKER> to your watchlist." plus one watchlist add |
| `remove <TICKER>` | Message "Removing <TICKER> from your watchlist." plus one watchlist remove |
| anything else | Message "This is a mock response from FinAlly." with no actions |

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. There are no client-side routes beyond `/`. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), change % since session open (`session_change_percent`), and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `/api/portfolio/history`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill. API errors are shown inline using the `error` message.
- **AI chat panel** — docked/collapsible sidebar. Loads `/api/chat/history` on page load. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations (failed actions shown in red with their error).
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Data Flow

- **Prices**: one `EventSource` on `/api/stream/prices` feeds the watchlist, sparklines, main chart, positions table and heatmap.
- **Portfolio**: fetched from `/api/portfolio` on load and refetched after every trade or chat response containing executed actions. No polling.
- **Live total value**: computed client-side as `cash_balance + Σ(quantity × latest SSE price)`, so the header updates with every price tick.
- **Connection status**: driven by `EventSource` events — `open` → green; `error` while `readyState === CONNECTING` → yellow; `readyState === CLOSED` → red.

### Technical Notes

- **Charts**: Lightweight Charts (TradingView, canvas-based) for the main chart and P&L chart. Sparklines are plain inline SVG polylines (no library). The heatmap is a small hand-rolled squarified treemap rendered as absolutely positioned divs, so there's no second charting library.
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000. The static mount is registered after all `/api` routers so it never shadows them.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start.ps1`** / **`scripts/stop.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface, session open price is recorded and kept
- Portfolio: trade execution logic, weighted-average cost, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss, selling the full position deletes the row, trading a ticker with no price)
- Tracked tickers: removing a held ticker from the watchlist keeps it priced; closing a position on an unwatched ticker stops tracking it
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow, watchlist changes applied before trades, mock-mode rules
- API routes: correct status codes, response shapes and error format (§8), app starts and chat returns `LLM_UNAVAILABLE` without an API key

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Connection status indicator transitions (connected → reconnecting → disconnected) driven by mocked `EventSource` events
- Watchlist CRUD operations
- Portfolio display calculations, including client-side live total value
- Chat message rendering, loading state, and failed-action display

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points (an initial snapshot exists from startup)
- AI chat (mocked): send "buy 1 AAPL", receive a response, trade execution appears inline; reload the page and the conversation is still shown

SSE reconnection is covered by the frontend unit test of the connection status indicator rather than E2E, because EventSource reconnection is browser-native and hard to test reliably against a single container.
