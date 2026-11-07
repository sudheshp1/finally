# FinAlly — AI Trading Workstation

FinAlly (Finance Ally) is an AI-powered trading workstation. It streams live market data, lets you trade a simulated $10,000 portfolio, and includes an AI chat assistant that can analyze your positions and place trades for you in plain English. Think of a Bloomberg terminal with an AI copilot.

It is the capstone project for an agentic AI coding course: the whole application is built by coding agents that coordinate through the documents in [`planning/`](planning/).

## Project Status

| Component | Status |
|---|---|
| Market data (simulator, Massive API client, price cache, SSE stream) | ✅ Built and tested — see [`planning/MARKET_DATA_SUMMARY.md`](planning/MARKET_DATA_SUMMARY.md) |
| Database, portfolio and watchlist API | ⏳ Planned |
| AI chat (LiteLLM → OpenRouter / Cerebras) | ⏳ Planned |
| Next.js frontend | ⏳ Planned |
| Dockerfile, start/stop scripts, E2E tests | ⏳ Planned |

The full specification is in [`planning/PLAN.md`](planning/PLAN.md). Review feedback on the plan is in [`planning/REVIEW.md`](planning/REVIEW.md).

## Features (Target)

- **Live prices** streamed over Server-Sent Events, flashing green or red on each change
- **Watchlist** with sparklines; add or remove tickers by hand or through the AI
- **Simulated trading**: market orders only, instant fills, fractional shares, no fees
- **Portfolio views**: a heatmap sized by weight and colored by P&L, a P&L chart and a positions table
- **AI assistant** that analyzes holdings, suggests trades and executes them automatically
- **Dark, data-dense terminal look**: no login, opens straight into the app

## Architecture

Everything runs in one Docker container on port 8000:

```
FastAPI (Python, uv)
├── /api/*          REST endpoints
├── /api/stream/*   SSE price stream
└── /*              Static Next.js export
SQLite (db/finally.db, volume-mounted)
Background tasks: market data simulator/poller, portfolio snapshots
```

| Layer | Technology |
|---|---|
| Frontend | Next.js + TypeScript (static export), Tailwind CSS, Lightweight Charts |
| Backend | FastAPI, managed with `uv` |
| Database | SQLite, created and seeded automatically on startup |
| AI | LiteLLM → OpenRouter (`openai/gpt-oss-120b` on Cerebras), structured outputs |
| Market data | Built-in GBM simulator (default) or Massive / Polygon.io REST API |

## What You Can Run Today

### Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/)

### Market Data Demo

A live terminal dashboard of simulated prices:

```bash
cd backend
uv sync --extra dev
uv run market_data_demo.py
```

### Tests

```bash
cd backend
uv run pytest              # run all tests
uv run pytest --cov=app    # with coverage
uv run ruff check .        # lint
```

## Planned Quick Start

Once the Docker build and scripts are in place:

```bash
cp .env.example .env        # optionally add OPENROUTER_API_KEY
./scripts/start.sh          # macOS/Linux
./scripts/start.ps1         # Windows PowerShell
```

Then open <http://localhost:8000>. Use `scripts/stop.sh` / `stop.ps1` to stop the container; your data is kept in a Docker volume.

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OPENROUTER_API_KEY` | No | Enables AI chat. Without it the app still runs; chat returns `LLM_UNAVAILABLE`. |
| `MASSIVE_API_KEY` | No | Real market data from Massive (Polygon.io). If unset, the built-in simulator is used. |
| `LLM_MOCK` | No | Set to `true` for deterministic mock chat responses (testing, no key needed). |

## Project Structure

```
finally/
├── backend/        # FastAPI uv project (market data built; API, DB and LLM to come)
│   ├── app/market/ # Simulator, Massive client, price cache, SSE stream
│   └── tests/      # pytest suite
├── planning/       # Specification and shared docs for the agents
├── .claude/        # Agent and command definitions used to build the project
├── frontend/       # (planned) Next.js app
├── scripts/        # (planned) start/stop scripts
├── test/           # (planned) Playwright E2E tests
└── db/             # (planned) SQLite volume mount
```

## License

MIT. See [LICENSE](LICENSE).
