# Market Data Backend — Detailed Design

**Scope:** The market data subsystem of FinAlly. It covers the unified data-source API, the shared price cache, the GBM simulator, the Massive (Polygon.io) REST poller, the SSE stream, and how the rest of the backend uses them.

**Sources read:** `PLAN.md`, `MARKET_DATA_SUMMARY.md`, and `archive/` (`MARKET_INTERFACE.md`, `MARKET_SIMULATOR.md`, `MASSIVE_API.md`, `MARKET_DATA_DESIGN.md`, `MARKET_DATA_REVIEW.md`). The design matches the code that currently lives in `backend/app/market/`.

**How to read this document.** Each section shows the reference implementation, which is already in the repo and tested. Sections marked **🔧 Proposed** describe changes this design recommends: closing gaps against `PLAN.md`, or fixing issues still open from the archived review. Agents building the portfolio, watchlist and chat features should code against the public API in §3 and the integration contracts in §11–§13.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Architecture](#2-architecture)
3. [Public API (what other modules import)](#3-public-api)
4. [File Layout](#4-file-layout)
5. [Data Model — `PriceUpdate`](#5-data-model--priceupdate)
6. [Price Cache — `PriceCache`](#6-price-cache--pricecache)
7. [Unified Interface — `MarketDataSource`](#7-unified-interface--marketdatasource)
8. [Simulator](#8-simulator)
9. [Massive API Client](#9-massive-api-client)
10. [Factory & Configuration](#10-factory--configuration)
11. [SSE Streaming Endpoint](#11-sse-streaming-endpoint)
12. [FastAPI Lifecycle Integration](#12-fastapi-lifecycle-integration)
13. [Downstream Consumers (watchlist, trades, portfolio, chat)](#13-downstream-consumers)
14. [Frontend Contract](#14-frontend-contract)
15. [Error Handling & Edge Cases](#15-error-handling--edge-cases)
16. [Testing](#16-testing)
17. [Proposed Changes Summary](#17-proposed-changes-summary)

---

## 1. Requirements

These requirements are taken from `PLAN.md` §6 and §10.

| # | Requirement | Where satisfied |
|---|---|---|
| R1 | Two implementations (simulator, Massive) behind one abstract interface | §7, §8, §9 |
| R2 | Source selected by `MASSIVE_API_KEY` (set and non-empty → Massive) | §10 |
| R3 | Simulator: GBM, per-ticker drift/vol, ~500 ms ticks, correlated moves, random 2–5 % events, realistic seed prices | §8 |
| R4 | Massive: REST polling of all watched tickers in one call; 15 s free tier, 2–15 s paid | §9 |
| R5 | Shared in-memory cache holding latest price, previous price and timestamp per ticker | §6 |
| R6 | `GET /api/stream/prices` SSE stream, ~500 ms cadence, each event carries ticker, price, previous price, timestamp, direction | §11 |
| R7 | Client reconnects automatically (EventSource retry) | §11 |
| R8 | Downstream code (SSE, portfolio, trades) does not care which source is active | §3, §13 |
| R9 | Watchlist shows **daily change %** | §5.2 🔧 |
| R10 | Tickers added or removed at runtime (manually or by the AI) start or stop streaming | §7, §13.1 |

---

## 2. Architecture

```
                 ┌────────────────────────────────────────┐
  env var  ───▶  │ create_market_data_source(cache)       │   factory.py
MASSIVE_API_KEY  └──────────────┬─────────────────────────┘
                                │ returns one of
             ┌──────────────────┴──────────────────┐
             ▼                                     ▼
  ┌─────────────────────┐              ┌─────────────────────┐
  │ SimulatorDataSource │              │ MassiveDataSource   │
  │  └ GBMSimulator     │              │  └ massive.RESTClient│
  │  asyncio task 0.5 s │              │  asyncio task 15 s  │
  └──────────┬──────────┘              └──────────┬──────────┘
             │  cache.update(ticker, price, ts)   │
             └──────────────────┬─────────────────┘
                                ▼
                 ┌──────────────────────────────┐
                 │ PriceCache (thread-safe)     │  cache.py
                 │  {ticker: PriceUpdate}       │
                 │  version: int (monotonic)    │
                 └──┬───────────┬───────────┬───┘
                    │           │           │   reads only
                    ▼           ▼           ▼
          SSE /api/stream   trade exec   portfolio valuation,
            /prices         (fill price) snapshots, LLM context
```

**Design principles**

1. **Strategy pattern.** There are two producers and one interface. Swapping data sources never touches consumer code.
2. **Push into a cache; never pull from the source.** Producers write to the cache on their own schedule. Consumers read whatever is newest, with no waiting. This makes SSE cadence independent of data cadence: the stream ticks every 500 ms whether data arrives every 0.5 s (simulator) or every 15 s (Massive).
3. **One process, one event loop.** Everything runs inside the FastAPI/uvicorn process as asyncio tasks. The only thread is the one `asyncio.to_thread` uses for the blocking Massive HTTP call. That thread is why the cache is lock-protected.
4. **Memory is bounded.** The cache holds only the latest `PriceUpdate` per ticker. History for sparklines and the main chart is built on the **frontend** from the stream (PLAN §2). Portfolio history lives in SQLite (`portfolio_snapshots`).

---

## 3. Public API

Everything outside `app/market/` should import only from the package root:

```python
from app.market import (
    PriceUpdate,               # immutable snapshot dataclass
    PriceCache,                # shared latest-price store
    MarketDataSource,          # ABC: start/stop/add_ticker/remove_ticker/get_tickers
    create_market_data_source, # factory: env-driven selection
    create_stream_router,      # FastAPI router for GET /api/stream/prices
)
```

`backend/app/market/__init__.py`:

```python
"""Market data subsystem for FinAlly."""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

**Typical consumer usage**

```python
price = cache.get_price("AAPL")        # float | None — for trade fills
update = cache.get("AAPL")             # PriceUpdate | None — for full detail
snapshot = cache.get_all()             # dict[str, PriceUpdate] — for valuation
await source.add_ticker("PYPL")        # watchlist add
await source.remove_ticker("PYPL")     # watchlist remove (also evicts cache)
```

---

## 4. File Layout

```
backend/
├── app/
│   └── market/
│       ├── __init__.py        # public re-exports (§3)
│       ├── models.py          # PriceUpdate (§5)
│       ├── cache.py           # PriceCache (§6)
│       ├── interface.py       # MarketDataSource ABC (§7)
│       ├── seed_prices.py     # seed prices, GBM params, correlation config (§8.2)
│       ├── simulator.py       # GBMSimulator + SimulatorDataSource (§8)
│       ├── massive_client.py  # MassiveDataSource (§9)
│       ├── factory.py         # create_market_data_source (§10)
│       └── stream.py          # SSE router (§11)
├── tests/market/              # pytest suites per module (§16)
└── market_data_demo.py        # Rich terminal dashboard of live simulated prices
```

Dependencies (in `backend/pyproject.toml`): `fastapi`, `uvicorn[standard]`, `numpy` (Cholesky, normal draws), `massive` (Polygon.io client). For development: `pytest`, `pytest-asyncio` (`asyncio_mode = "auto"`), `pytest-cov`, `ruff`.

---

## 5. Data Model — `PriceUpdate`

### 5.1 Reference implementation (`models.py`)

`PriceUpdate` is the only type that leaves the market layer. It is frozen, so it can be shared between the cache, SSE serialisation and request handlers with no copying and no risk of mutation.

```python
from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

Derived fields (`change`, `change_percent`, `direction`) are computed properties rather than stored fields, so they can never disagree with `price` and `previous_price`.

**Example**

```python
>>> u = PriceUpdate("AAPL", price=190.42, previous_price=190.17, timestamp=1762700000.0)
>>> u.direction, u.change, u.change_percent
('up', 0.25, 0.1315)
>>> u.to_dict()
{'ticker': 'AAPL', 'price': 190.42, 'previous_price': 190.17, 'timestamp': 1762700000.0,
 'change': 0.25, 'change_percent': 0.1315, 'direction': 'up'}
```

### 5.2 🔧 Proposed: daily change (session reference price)

`change_percent` is **tick-over-tick**: the change since the previous update. That is the right value for the flash animation. But PLAN §10 asks the watchlist to show **daily change %**, and with a tick-over-tick value the watchlist would show figures like `0.0012 %`. The fix is to add a per-ticker **session reference price**:

| Source | `session_open` value |
|---|---|
| Simulator | The first price written for the ticker in this process (its seed price, or the price when it was added) |
| Massive | `snap.day.previous_close` (the standard "day change" baseline). If it is missing, fall back to the first observed price |

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)
    session_open: float | None = None          # NEW: baseline for daily change

    # ... change / change_percent / direction unchanged ...

    @property
    def day_change(self) -> float:
        base = self.session_open if self.session_open is not None else self.previous_price
        return round(self.price - base, 4)

    @property
    def day_change_percent(self) -> float:
        base = self.session_open if self.session_open is not None else self.previous_price
        if base == 0:
            return 0.0
        return round((self.price - base) / base * 100, 4)

    def to_dict(self) -> dict:
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
            "session_open": self.session_open,          # NEW
            "day_change": self.day_change,              # NEW
            "day_change_percent": self.day_change_percent,  # NEW
        }
```

The new field has a default, so this change is backward compatible: existing constructor calls and tests keep working. The cache supplies the value (see §6.3).

---

## 6. Price Cache — `PriceCache`

### 6.1 Reference implementation (`cache.py`)

```python
from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price   # first write → flat

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        with self._lock:
            return dict(self._prices)       # shallow copy; values are immutable

    def get_price(self, ticker: str) -> float | None:
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

### 6.2 Design notes

- **Why `threading.Lock` and not `asyncio.Lock`?** Writes can come from a worker thread (Massive runs inside `asyncio.to_thread`), and readers may be sync functions. A plain mutex works from both worlds. Each critical section is only a few dictionary operations, so it never blocks the event loop in any noticeable way.
- **`previous_price` is the previous *cache write*, not the previous SSE frame.** For the simulator the two are the same. For Massive, `previous_price` is the price from the previous 15 s poll, so the flash direction shows how the price moved between polls.
- **Rounding to 2 dp happens at the cache.** The simulator keeps full precision internally (§8), so rounding does not affect the GBM random walk.
- **`version`** is a monotonic counter bumped on every write. SSE generators compare it with the last value they sent and skip unchanged frames (§11). This is cheaper than diffing dictionaries, and with N clients it costs N integer comparisons per tick.

### 6.3 🔧 Proposed changes

1. **Session baseline for daily change (§5.2).** The cache remembers the first price per ticker. A source can override it (Massive passes `previous_close`).
2. **Read `version` under the lock.** This is review item 3.4. The unlocked read is atomic only because of the GIL, and Python 3.13+ free-threaded builds remove that guarantee.

```python
class PriceCache:
    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._session_open: dict[str, float] = {}   # NEW
        self._lock = Lock()
        self._version = 0

    def update(
        self,
        ticker: str,
        price: float,
        timestamp: float | None = None,
        session_open: float | None = None,           # NEW (Massive passes previous_close)
    ) -> PriceUpdate:
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price
            if session_open is not None:
                self._session_open[ticker] = session_open
            base = self._session_open.setdefault(ticker, price)

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
                session_open=round(base, 2),
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)
            self._session_open.pop(ticker, None)     # NEW

    @property
    def version(self) -> int:
        with self._lock:                             # NEW
            return self._version
```

---

## 7. Unified Interface — `MarketDataSource`

### 7.1 Reference implementation (`interface.py`)

```python
from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Starts a background task. Call once."""

    @abstractmethod
    async def stop(self) -> None:
        """Cancel the background task. Idempotent. No cache writes afterwards."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set and evict it from the cache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Current actively tracked tickers."""
```

### 7.2 Contract details every implementation must honour

| Method | Behaviour |
|---|---|
| `start(tickers)` | Must put **at least one price per ticker into the cache before returning**, where possible, so the first SSE frame is complete. The simulator writes its seed prices. Massive does one synchronous poll first. Then it spawns a named `asyncio.Task`. |
| `stop()` | Cancels the task, awaits it, swallows `CancelledError`. Safe to call twice, or before `start`. |
| `add_ticker(t)` | Idempotent. The simulator writes a price immediately. Massive picks the ticker up on the next poll (≤ interval). |
| `remove_ticker(t)` | Idempotent. Always calls `cache.remove(t)` so the ticker disappears from the next SSE frame. |
| `get_tickers()` | Returns a **copy**. |
| Loop resilience | The background loop must never die from one bad tick or poll. Catch, log, continue. |

**Ticker normalisation.** Callers (the watchlist API and the LLM tool layer) must pass upper-case, stripped symbols. `MassiveDataSource` also normalises defensively. 🔧 *Proposed:* `SimulatorDataSource.add_ticker`/`remove_ticker` should apply the same `ticker.upper().strip()`, so both sources behave identically.

### 7.3 Lifecycle

```python
cache = PriceCache()
source = create_market_data_source(cache)          # unstarted
await source.start(["AAPL", "GOOGL", "MSFT"])      # cache seeded, task running
await source.add_ticker("TSLA")
await source.remove_ticker("GOOGL")
await source.stop()                                 # task cancelled
```

---

## 8. Simulator

The simulator has two layers. `GBMSimulator` is pure synchronous maths, deterministic under a seeded RNG and easy to unit-test. `SimulatorDataSource` is the async adapter that implements `MarketDataSource`.

### 8.1 The maths

Geometric Brownian Motion, exact discretisation (log-normal, so prices never go negative):

```
S(t+dt) = S(t) · exp( (μ − σ²/2)·dt + σ·√dt·Z )
```

- `μ`: annualised drift. `σ`: annualised volatility.
- `dt`: one tick as a fraction of a trading year: `0.5 s / (252 · 6.5 · 3600 s) ≈ 8.48e-8`.
- `Z`: a **correlated** standard normal, `Z = L · z`, where `z ~ N(0, I)` and `L` is the Cholesky factor of the correlation matrix `C = L·Lᵀ`.

**Size of moves.** The per-tick standard deviation is `σ·√dt`. For AAPL (σ = 0.22) that is ≈ 0.0064 %, about 1.2 ¢ per tick on a $190 stock. Over one simulated trading day the moves add up to a realistic ~1.4 % daily standard deviation, but a full trading day is 46,800 ticks (6.5 h), so in a short demo prices look calm. The random events add visible drama.

**Random events.** Each tick, each ticker has probability `p = 0.001` of a shock of ±U(2 %, 5 %). With 10 tickers at 2 ticks/s, that gives one event roughly every 50 s.

### 8.2 Parameters (`seed_prices.py`)

```python
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00, "TSLA": 250.00,
    "NVDA": 800.00, "META": 500.00, "JPM": 195.00, "V": 280.00, "NFLX": 600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},   # high vol
    "NVDA": {"sigma": 0.40, "mu": 0.08},   # high vol, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM":  {"sigma": 0.18, "mu": 0.04},   # low vol (bank)
    "V":    {"sigma": 0.17, "mu": 0.04},   # low vol (payments)
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}   # dynamically added tickers

CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}
INTRA_TECH_CORR = 0.6
INTRA_FINANCE_CORR = 0.5
CROSS_GROUP_CORR = 0.3
TSLA_CORR = 0.3
```

A ticker that has no seed price (for example, `PYPL` added by the AI) starts at `random.uniform(50, 300)` and uses `DEFAULT_PARAMS`.

**Correlation rules** (`_pairwise_correlation`):

| Pair | ρ |
|---|---|
| Either side is TSLA | 0.3 (TSLA is listed in tech but moves on its own) |
| Both tech | 0.6 |
| Both finance | 0.5 |
| Anything else (cross-sector, unknown tickers) | 0.3 |

This matrix is always positive-definite for any set of tickers. It is equicorrelated blocks of ρ ≤ 0.6 on a 0.3 floor, so `np.linalg.cholesky` cannot fail as tickers are added.

### 8.3 `GBMSimulator` (`simulator.py`)

```python
class GBMSimulator:
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600      # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR      # ~8.48e-8

    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}           # full precision
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None
        for ticker in tickers:
            self._add_ticker_internal(ticker)          # batch add …
        self._rebuild_cholesky()                       # … one O(n²) rebuild

    def step(self) -> dict[str, float]:
        """Advance all tickers one tick. Hot path (every 500 ms)."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z = np.random.standard_normal(n)
        if self._cholesky is not None:
            z = self._cholesky @ z                     # correlate

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu, sigma = self._params[ticker]["mu"], self._params[ticker]["sigma"]
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            if random.random() < self._event_prob:     # drama
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock

            result[ticker] = round(self._prices[ticker], 2)
        return result

    def add_ticker(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    def _add_ticker_internal(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = corr[j, i] = rho
        self._cholesky = np.linalg.cholesky(corr)
```

**Why keep full precision internally?** If the simulator rounded to cents every tick, sub-cent moves would be cut off and prices would get "stuck" on low-volatility names. The simulator rounds only the value it returns. The internal state stays exact.

**Determinism for tests:** call `np.random.seed(…)` and `random.seed(…)` before constructing the simulator, or pass `event_probability=0.0` to remove shocks.

### 8.4 `SimulatorDataSource`

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        for ticker in tickers:                          # seed cache → first SSE frame is full
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)   # visible immediately

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    for ticker, price in self._sim.step().items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")   # never let the loop die
            await asyncio.sleep(self._interval)
```

**Concurrency note.** `add_ticker`, `remove_ticker` and `step()` all run on the event loop thread, and `step()` never `await`s in the middle. The simulator's lists and dictionaries therefore cannot be modified halfway through a step, so the simulator needs no lock of its own.

---

## 9. Massive API Client

### 9.1 Endpoint

Each poll makes exactly one call, whatever the number of tickers:

```
GET https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT
Authorization: Bearer $MASSIVE_API_KEY
```

The `massive` Python package wraps this as `RESTClient.get_snapshot_all(market_type=SnapshotMarketType.STOCKS, tickers=[...])`. Fields used from each snapshot:

| Field | Use |
|---|---|
| `snap.ticker` | cache key |
| `snap.last_trade.price` | current price |
| `snap.last_trade.timestamp` | Unix **milliseconds** → divide by 1000 |
| `snap.day.previous_close` | 🔧 session baseline for daily change (§5.2) |

An excerpt of the response for one ticker:

```json
{
  "ticker": "AAPL",
  "day": {"open": 129.61, "high": 130.15, "low": 125.07, "close": 125.07,
          "volume": 111237700, "previous_close": 129.61, "change_percent": -3.50},
  "last_trade": {"price": 125.07, "size": 100, "exchange": "XNYS", "timestamp": 1675190399000}
}
```

### 9.2 Rate limits → poll interval

| Tier | Limit | Recommended interval |
|---|---|---|
| Free | 5 req/min | **15 s** (default, 4 req/min leaves headroom) |
| Paid | effectively unlimited | 2–5 s |

### 9.3 Reference implementation (`massive_client.py`)

```python
import asyncio
import logging

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    def __init__(self, api_key: str, price_cache: PriceCache,
                 poll_interval: float = 15.0) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()                      # cache populated before first SSE frame
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)             # appears on next poll

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # RESTClient is synchronous (urllib3) → run off the event loop.
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    self._cache.update(
                        ticker=snap.ticker,
                        price=snap.last_trade.price,
                        timestamp=snap.last_trade.timestamp / 1000.0,   # ms → s
                    )
                except (AttributeError, TypeError) as e:
                    # e.g. a ticker with no trades yet → last_trade is None
                    logger.warning("Skipping snapshot for %s: %s",
                                   getattr(snap, "ticker", "???"), e)
        except Exception as e:
            # 401 bad key, 403 plan, 429 rate limit, network errors.
            # Log and retry on the next interval; the cache keeps last-known prices.
            logger.error("Massive poll failed: %s", e)

    def _fetch_snapshots(self) -> list:
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### 9.4 Behaviour notes

- **The first poll happens inside `start()`.** If it fails (for example, a bad key), `start()` still returns and the app still boots. The cache stays empty and SSE sends nothing until a poll succeeds. The error log makes the cause obvious.
- **Market closed.** `last_trade.price` is the last traded price (it may be after-hours), and the timestamp stops moving. The cache is still updated each poll with the same price, so `direction` becomes `"flat"` and `version` keeps rising. SSE frames keep flowing, but nothing flashes. This is correct behaviour.
- **Unknown / invalid symbols.** The snapshot endpoint simply leaves them out of the response, so they never enter the cache (see §13.1 for the API response).
- **SSE vs poll cadence.** SSE still emits every 500 ms, but only when `version` has changed. With Massive that means one frame per poll.

### 9.5 🔧 Proposed changes

1. **Configurable interval.** Read `MASSIVE_POLL_INTERVAL` (seconds, default 15, minimum 2) in the factory, so paid-tier users get faster updates without a code change.
2. **Pass the daily-change baseline:**

   ```python
   prev_close = getattr(getattr(snap, "day", None), "previous_close", None)
   self._cache.update(
       ticker=snap.ticker,
       price=snap.last_trade.price,
       timestamp=snap.last_trade.timestamp / 1000.0,
       session_open=prev_close or None,       # 0/None → cache falls back to first price
   )
   ```
3. **Back off on 429.** After a rate-limit error, double the sleep (capped at 120 s) and reset it after the next success. This keeps a free-tier key from staying throttled:

   ```python
   async def _poll_loop(self) -> None:
       delay = self._interval
       while True:
           await asyncio.sleep(delay)
           ok = await self._poll_once()            # make _poll_once return bool
           delay = self._interval if ok else min(delay * 2, 120.0)
   ```
4. **Immediate first price for added tickers.** In `add_ticker`, trigger `asyncio.create_task(self._poll_once())`, but only on paid tiers (interval < 15 s). On the free tier an extra call could exceed 5 req/min.

---

## 10. Factory & Configuration

### 10.1 Reference implementation (`factory.py`)

```python
import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """MASSIVE_API_KEY set and non-empty → Massive; otherwise → simulator.
    Returns an unstarted source; caller must await source.start(tickers)."""
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    logger.info("Market data source: GBM Simulator")
    return SimulatorDataSource(price_cache=price_cache)
```

The `.strip()` matters. `MASSIVE_API_KEY=` and `MASSIVE_API_KEY="  "` in `.env` both select the simulator, as PLAN §5 requires.

### 10.2 🔧 Proposed: interval override

```python
def _float_env(name: str, default: float, minimum: float) -> float:
    try:
        return max(minimum, float(os.environ.get(name, default)))
    except ValueError:
        logger.warning("Invalid %s; using %.1f", name, default)
        return default

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        interval = _float_env("MASSIVE_POLL_INTERVAL", 15.0, minimum=2.0)
        return MassiveDataSource(api_key=api_key, price_cache=price_cache,
                                 poll_interval=interval)
    return SimulatorDataSource(price_cache=price_cache)
```

### 10.3 Configuration summary

| Setting | Where | Default | Notes |
|---|---|---|---|
| `MASSIVE_API_KEY` | env | empty | non-empty → Massive |
| `MASSIVE_POLL_INTERVAL` | env 🔧 | 15.0 | seconds, min 2 |
| Simulator tick | `SimulatorDataSource(update_interval=)` | 0.5 s | |
| Event probability | `SimulatorDataSource(event_probability=)` | 0.001 | per ticker per tick |
| SSE cadence | `_generate_events(interval=)` | 0.5 s | |
| SSE retry | `retry:` directive | 1000 ms | |

---

## 11. SSE Streaming Endpoint

### 11.1 Wire format

Every frame is a **single event holding the full snapshot** of all tracked tickers. The frontend replaces its price map wholesale, so no delta logic is needed and a reconnecting client recovers in one frame.

```
retry: 1000

data: {"AAPL":{"ticker":"AAPL","price":190.42,"previous_price":190.17,"timestamp":1762700000.51,"change":0.25,"change_percent":0.1315,"direction":"up"},"GOOGL":{...}, ...}

data: {...}

```

- The first line, `retry: 1000`, tells `EventSource` to reconnect 1 s after a drop.
- A frame is sent only when `cache.version` has changed since the last frame for this client.
- Payload size is about 10 tickers × ~170 B ≈ 1.7 KB per frame, or roughly 3.4 KB/s per client. That is negligible.

### 11.2 Reference implementation (`stream.py`)

```python
import asyncio
import json
import logging
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/stream", tags=["streaming"])


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",
            },
        )
    return router


async def _generate_events(price_cache: PriceCache, request: Request,
                           interval: float = 0.5) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"
    last_version = -1
    try:
        while True:
            if await request.is_disconnected():
                break
            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()
                if prices:
                    data = {t: u.to_dict() for t, u in prices.items()}
                    yield f"data: {json.dumps(data)}\n\n"
            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled")
```

**Why poll the cache instead of using an event-driven fan-out (asyncio.Queue per client)?** Polling is stateless per client, needs no subscriber bookkeeping, sets a natural maximum send rate (2 Hz) however fast producers write, and handles slow clients gracefully because they just get the latest snapshot. With one user and ten tickers, the cost is irrelevant.

### 11.3 🔧 Proposed: create the router inside the factory

This is review item 3.6, which is still open. The module-level `router` means that calling `create_stream_router()` twice (for example, in tests that build several apps) registers `/prices` twice on the same object. The fix is to construct the router inside the function:

```python
def create_stream_router(price_cache: PriceCache) -> APIRouter:
    router = APIRouter(prefix="/api/stream", tags=["streaming"])   # moved inside

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        ...

    return router
```

Optional: send a comment heartbeat (`": ping\n\n"`) every ~15 s when nothing has changed (for example, Massive with the market closed). This stops idle-timeout proxies (App Runner, Render) from closing the connection.

```python
            elif loop.time() - last_sent > 15:
                yield ": ping\n\n"
                last_sent = loop.time()
```

---

## 12. FastAPI Lifecycle Integration

The market layer is created once in the app's `lifespan` and stored on `app.state`. Route modules get it through small dependency functions, so nothing uses module-level globals.

```python
# backend/app/main.py
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request

from app.db import init_db, get_watchlist_tickers          # owned by the backend agent
from app.market import PriceCache, create_market_data_source, create_stream_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()                                               # lazy schema + seed
    cache = PriceCache()
    source = create_market_data_source(cache)
    await source.start(get_watchlist_tickers(user_id="default"))

    app.state.price_cache = cache
    app.state.market_source = source
    app.include_router(create_stream_router(cache))

    snapshot_task = asyncio.create_task(record_snapshots_forever(cache))  # §13.3
    try:
        yield
    finally:
        snapshot_task.cancel()
        await source.stop()


app = FastAPI(lifespan=lifespan)


# --- dependencies used by route modules ---
def get_price_cache(request: Request) -> PriceCache:
    return request.app.state.price_cache

def get_market_source(request: Request) -> MarketDataSource:
    return request.app.state.market_source
```

> If including the router inside `lifespan` causes trouble with how the static-files mount is ordered, create the `PriceCache` at module import time instead. Include the router at import time, and start only the *source* in `lifespan`. The cache has no async state, so creating it early is safe. Mount the static frontend **last** so `/api/*` routes take precedence.

**Startup ordering:** DB init → read the watchlist → `source.start(tickers)` (seeds the cache) → serve. When the first request arrives, prices already exist.

---

## 13. Downstream Consumers

### 13.1 Watchlist coordination

The DB watchlist is the source of truth for **what** is tracked. The data source is told about each change. The market data source must track the union of **watchlist tickers ∪ tickers with open positions**. Otherwise a position in a ticker the user has unwatched could not be valued or sold.

```python
# backend/app/routes/watchlist.py
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(prefix="/api/watchlist")

@router.get("")
async def list_watchlist(cache: PriceCache = Depends(get_price_cache)):
    tickers = db.get_watchlist_tickers("default")
    return [
        {"ticker": t, **(cache.get(t).to_dict() if cache.get(t) else {"price": None})}
        for t in tickers
    ]

@router.post("", status_code=201)
async def add_to_watchlist(body: TickerIn,
                           source: MarketDataSource = Depends(get_market_source)):
    ticker = body.ticker.upper().strip()
    if not TICKER_RE.fullmatch(ticker):                    # r"[A-Z][A-Z0-9.\-]{0,9}"
        raise HTTPException(400, "Invalid ticker symbol")
    db.add_watchlist_ticker("default", ticker)            # UNIQUE → idempotent
    await source.add_ticker(ticker)
    return {"ticker": ticker}

@router.delete("/{ticker}")
async def remove_from_watchlist(ticker: str,
                                source: MarketDataSource = Depends(get_market_source)):
    ticker = ticker.upper().strip()
    db.remove_watchlist_ticker("default", ticker)
    if not db.has_position("default", ticker):             # keep pricing held positions
        await source.remove_ticker(ticker)
    return {"ticker": ticker, "removed": True}
```

On startup, call `source.start(sorted(set(watchlist) | set(position_tickers)))`. After a sell that closes a position in an unwatched ticker, call `source.remove_ticker`.

**Invalid symbols:** with the simulator, any well-formed symbol "exists" and gets a random seed price. With Massive, an unknown symbol never appears in the cache, so the watchlist shows `price: null`. The frontend should render that as "—".

### 13.2 Trade execution (fill price)

Trades fill at the cached price. With no cached price the trade is rejected. The fill price is never invented.

```python
def execute_trade(ticker: str, side: str, quantity: float, cache: PriceCache) -> Trade:
    ticker = ticker.upper().strip()
    price = cache.get_price(ticker)
    if price is None:
        raise TradeError(f"No live price for {ticker}; add it to the watchlist first")
    if quantity <= 0:
        raise TradeError("Quantity must be positive")
    # ... cash / share validation, update positions + trades in one DB transaction ...
    # after a BUY of a ticker not yet tracked (can't happen if price exists) — n/a
    # after commit: record a portfolio snapshot (PLAN §7)
```

The same function serves `POST /api/portfolio/trade` and the LLM auto-execution path. If a chat trade targets an untracked ticker, the chat handler should first `await source.add_ticker(t)`. With the simulator that makes the price available at once. With Massive the trade fails with the message above, and the LLM reports it.

### 13.3 Portfolio valuation & snapshots

```python
def portfolio_value(positions: list[Position], cash: float, cache: PriceCache) -> dict:
    prices = cache.get_all()                               # one lock, consistent snapshot
    rows, total = [], cash
    for p in positions:
        current = prices[p.ticker].price if p.ticker in prices else p.avg_cost  # fallback
        mv = current * p.quantity
        total += mv
        rows.append({
            "ticker": p.ticker, "quantity": p.quantity, "avg_cost": p.avg_cost,
            "current_price": current,
            "unrealized_pnl": round((current - p.avg_cost) * p.quantity, 2),
            "pnl_percent": round((current / p.avg_cost - 1) * 100, 2) if p.avg_cost else 0.0,
        })
    return {"cash": cash, "positions": rows, "total_value": round(total, 2)}


async def record_snapshots_forever(cache: PriceCache, every: float = 30.0) -> None:
    while True:
        await asyncio.sleep(every)
        try:
            v = portfolio_value(db.get_positions("default"), db.get_cash("default"), cache)
            db.insert_snapshot("default", v["total_value"])
        except Exception:
            logger.exception("snapshot failed")
```

### 13.4 LLM context

The chat handler builds its portfolio context from the same `portfolio_value(...)`, plus the watchlist with `cache.get(t).to_dict()`. The LLM therefore sees exactly the prices the UI shows.

---

## 14. Frontend Contract

TypeScript types that mirror `PriceUpdate.to_dict()`:

```ts
export type Direction = "up" | "down" | "flat";

export interface PriceUpdate {
  ticker: string;
  price: number;
  previous_price: number;
  timestamp: number;          // Unix seconds (float)
  change: number;             // vs previous tick
  change_percent: number;     // vs previous tick
  direction: Direction;
  // 🔧 proposed (§5.2):
  session_open?: number | null;
  day_change?: number;
  day_change_percent?: number;
}

export type PriceSnapshot = Record<string, PriceUpdate>;
```

A minimal hook covering connection status, price map, sparkline history and flash triggers:

```ts
import { useEffect, useRef, useState } from "react";

type Status = "connected" | "reconnecting" | "disconnected";
const MAX_POINTS = 300;

export function usePriceStream() {
  const [prices, setPrices] = useState<PriceSnapshot>({});
  const [status, setStatus] = useState<Status>("reconnecting");
  const history = useRef<Record<string, { t: number; p: number }[]>>({});

  useEffect(() => {
    const es = new EventSource("/api/stream/prices");
    es.onopen = () => setStatus("connected");
    es.onerror = () =>
      setStatus(es.readyState === EventSource.CLOSED ? "disconnected" : "reconnecting");
    es.onmessage = (ev) => {
      const snap: PriceSnapshot = JSON.parse(ev.data);
      for (const u of Object.values(snap)) {
        const h = (history.current[u.ticker] ??= []);
        if (h.at(-1)?.t !== u.timestamp) {
          h.push({ t: u.timestamp, p: u.price });
          if (h.length > MAX_POINTS) h.shift();
        }
      }
      setPrices(snap);
    };
    return () => es.close();
  }, []);

  return { prices, status, history: history.current };
}
```

Flash rule: when a ticker's `price` differs from the value rendered last time, apply `flash-up` or `flash-down` (based on `direction`) and remove it after ~500 ms. Deduplicating history by `timestamp` stops repeated Massive frames, or flat frames with the market closed, from padding the sparkline.

---

## 15. Error Handling & Edge Cases

| Scenario | Behaviour |
|---|---|
| Empty watchlist at startup | `start([])` is valid. The simulator steps nothing, the Massive poll is skipped, SSE sends no frames until a ticker is added. |
| Simulator tick raises | Logged with traceback. The loop continues on the next tick. |
| Massive 401 / 403 | Logged each poll. The cache keeps any last-known prices. The app keeps running. |
| Massive 429 | Logged. Retried next interval (🔧 exponential backoff, §9.5). |
| Malformed snapshot (no `last_trade`) | That ticker is skipped with a warning. The others are still updated. |
| Ticker removed while a trade is in flight | `get_price` returns `None` and the trade is rejected cleanly. |
| Price lookup for an unknown ticker | `None`, never an exception. The caller decides (reject trade, fall back to `avg_cost` for valuation). |
| Client disconnects | `request.is_disconnected()` ends the generator within ≤ 0.5 s. Cancellation is handled. |
| Network drop / server restart | `EventSource` reconnects after 1 s (`retry:`). The first frame is a full snapshot. |
| Many tickers added | Cholesky rebuild is O(n²) per add and stays negligible below a few hundred tickers. Massive allows one call for any list length. |
| Shutdown | `lifespan` → `source.stop()` cancels the task. There are no orphaned threads (an in-flight `to_thread` call finishes and its result is discarded). |

---

## 16. Testing

Existing suites in `backend/tests/market/`: `test_models.py`, `test_cache.py`, `test_simulator.py`, `test_simulator_source.py`, `test_massive.py`, `test_factory.py`. Run them with:

```bash
cd backend && uv run --extra dev pytest -v
```

```bash
cd backend && uv run --extra dev pytest --cov=app
```

Representative examples of each kind of test follow.

**GBM correctness (statistical)**

```python
import numpy as np, random
from app.market.simulator import GBMSimulator

def test_log_returns_match_sigma():
    np.random.seed(0); random.seed(0)
    sim = GBMSimulator(["AAPL"], event_probability=0.0)
    prices = [sim.get_price("AAPL")]
    for _ in range(20_000):
        sim.step(); prices.append(sim.get_price("AAPL"))
    r = np.diff(np.log(prices))
    expected = 0.22 * np.sqrt(GBMSimulator.DEFAULT_DT)
    assert abs(r.std() / expected - 1) < 0.05

def test_full_default_set_cholesky_ok():
    from app.market.seed_prices import SEED_PRICES
    sim = GBMSimulator(list(SEED_PRICES))
    assert sim._cholesky.shape == (10, 10)
    assert all(p > 0 for p in sim.step().values())
```

**Cache semantics**

```python
def test_first_update_is_flat_then_direction():
    c = PriceCache()
    assert c.update("AAPL", 100.0).direction == "flat"
    u = c.update("AAPL", 101.0)
    assert (u.direction, u.previous_price, u.change) == ("up", 100.0, 1.0)

def test_version_bumps_and_remove():
    c = PriceCache(); v0 = c.version
    c.update("X", 1.0); assert c.version == v0 + 1
    c.remove("X"); assert c.get("X") is None
```

**Simulator data source (async)**

```python
async def test_start_seeds_cache_and_ticks():
    cache = PriceCache()
    src = SimulatorDataSource(cache, update_interval=0.05)
    await src.start(["AAPL", "MSFT"])
    assert cache.get_price("AAPL") == 190.00          # seeded before first tick
    v = cache.version
    await asyncio.sleep(0.2)
    assert cache.version > v
    await src.remove_ticker("MSFT")
    assert "MSFT" not in cache
    await src.stop(); await src.stop()                # idempotent
```

**Massive (mocked, no network)**

```python
from unittest.mock import MagicMock, patch

def snap(t, p, ms):
    s = MagicMock(); s.ticker = t; s.last_trade.price = p; s.last_trade.timestamp = ms
    return s

async def test_poll_converts_ms_and_skips_bad():
    cache = PriceCache()
    src = MassiveDataSource("k", cache, poll_interval=60)
    src._tickers, src._client = ["AAPL", "BAD"], MagicMock()
    bad = MagicMock(); bad.ticker = "BAD"; bad.last_trade = None
    with patch.object(src, "_fetch_snapshots",
                      return_value=[snap("AAPL", 190.5, 1_700_000_000_000), bad]):
        await src._poll_once()
    assert cache.get("AAPL").timestamp == 1_700_000_000.0
    assert "BAD" not in cache

async def test_api_error_does_not_raise():
    src = MassiveDataSource("k", PriceCache(), poll_interval=60)
    src._tickers, src._client = ["AAPL"], MagicMock()
    with patch.object(src, "_fetch_snapshots", side_effect=RuntimeError("429")):
        await src._poll_once()                        # logged, swallowed
```

**Factory**

```python
def test_factory_selects(monkeypatch):
    monkeypatch.setenv("MASSIVE_API_KEY", "  ")
    assert isinstance(create_market_data_source(PriceCache()), SimulatorDataSource)
    monkeypatch.setenv("MASSIVE_API_KEY", "abc")
    assert isinstance(create_market_data_source(PriceCache()), MassiveDataSource)
```

**🔧 Missing: SSE integration test.** `stream.py` has low coverage. Exercise the generator directly with a fake request, which avoids needing a live server:

```python
class FakeRequest:
    client = None
    def __init__(self): self.n = 0
    async def is_disconnected(self):
        self.n += 1
        return self.n > 2

async def test_sse_emits_retry_then_snapshot():
    from app.market.stream import _generate_events
    cache = PriceCache(); cache.update("AAPL", 190.0)
    frames = [f async for f in _generate_events(cache, FakeRequest(), interval=0.01)]
    assert frames[0] == "retry: 1000\n\n"
    assert frames[1].startswith("data: ") and '"AAPL"' in frames[1]
    assert len(frames) == 2          # second loop: version unchanged → no duplicate frame
```

**E2E (`test/`, Playwright, `LLM_MOCK=true`):** these use the simulator, because the test compose file leaves `MASSIVE_API_KEY` unset. Assert that 10 tickers render, that a price cell changes within 2 s, that the connection dot turns green, and that adding `PYPL` makes it stream.

---

## 17. Proposed Changes Summary

None of these change the public API in a breaking way.

| # | Change | Files | Why |
|---|---|---|---|
| P1 | `session_open` + `day_change` / `day_change_percent` on `PriceUpdate`. The cache tracks the baseline. Massive passes `previous_close`. | `models.py`, `cache.py`, `massive_client.py` | PLAN §10 asks for **daily** change % in the watchlist (R9) |
| P2 | Build the router inside `create_stream_router` | `stream.py` | Review 3.6: route registered twice if the factory is called twice |
| P3 | Read `PriceCache.version` under the lock | `cache.py` | Review 3.4: correctness on free-threaded Python |
| P4 | `MASSIVE_POLL_INTERVAL` env override (min 2 s) | `factory.py`, `.env.example` | PLAN §6: paid tiers poll every 2–15 s |
| P5 | 429 exponential backoff (`_poll_once` returns bool) | `massive_client.py` | Recover from free-tier throttling |
| P6 | Normalise tickers (`upper().strip()`) in `SimulatorDataSource` too | `simulator.py` | Both sources behave the same |
| P7 | SSE heartbeat comment every 15 s when idle | `stream.py` | Keeps connections open behind cloud proxies when the market is closed |
| P8 | Track watchlist ∪ open positions | watchlist / portfolio routes (§13.1) | Held positions stay priced after an unwatch |
| P9 | SSE generator unit test | `tests/market/test_stream.py` | Covers the main consumer of the cache |

Implementation order: P2, P3, P6, P9 (small fixes) → P1 (needed before the frontend builds the watchlist) → P4, P5, P7 → P8 (built together with the watchlist and portfolio routes).
