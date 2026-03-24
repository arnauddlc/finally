# Market Data Subsystem — Implementation Summary

**Status: Complete**

The market data subsystem lives in `backend/app/market/`. It provides live (simulated or real) price data to the rest of the application via an in-memory cache and an SSE endpoint.

---

## Architecture

```
backend/app/market/
├── __init__.py          # Public API exports
├── interface.py         # Abstract MarketDataSource base class
├── models.py            # PriceUpdate immutable dataclass
├── cache.py             # Thread-safe PriceCache
├── seed_prices.py       # Seed prices and GBM params for default tickers
├── simulator.py         # GBMSimulator + SimulatorDataSource
├── massive_client.py    # MassiveDataSource (Polygon.io REST API)
├── factory.py           # create_market_data_source factory function
└── stream.py            # FastAPI SSE router for /api/stream/prices
```

---

## Components

### `MarketDataSource` (interface.py)
Abstract base class defining the contract for all market data providers:
- `start(tickers)` — begin producing price updates
- `stop()` — stop the background task
- `add_ticker(ticker)` — add a ticker to the active set
- `remove_ticker(ticker)` — remove a ticker and clear from cache
- `get_tickers()` — return the current list of tracked tickers

### `PriceUpdate` (models.py)
Immutable frozen dataclass representing a single price snapshot:
- `ticker`, `price`, `previous_price`, `timestamp`
- Computed properties: `change`, `change_percent`, `direction` ("up"/"down"/"flat")
- `to_dict()` — JSON-serializable dict for SSE transmission

### `PriceCache` (cache.py)
Thread-safe in-memory store for the latest price per ticker:
- `update(ticker, price, timestamp)` → `PriceUpdate`
- `get(ticker)` → `PriceUpdate | None`
- `get_all()` → `dict[str, PriceUpdate]`
- `get_price(ticker)` → `float | None`
- `remove(ticker)` — evict a ticker from cache
- `version` property — monotonic counter, increments on every update (used by SSE for change detection)

### `GBMSimulator` + `SimulatorDataSource` (simulator.py)
Default implementation using Geometric Brownian Motion:

**GBM Math**: `S(t+dt) = S(t) * exp((mu - σ²/2)*dt + σ*sqrt(dt)*Z)`

Where `Z` is drawn from a correlated multivariate normal via Cholesky decomposition of the correlation matrix. This produces realistic correlated price moves across tickers.

**Correlation structure:**
- Same tech group (AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX): ρ = 0.6
- Same finance group (JPM, V): ρ = 0.5
- TSLA with anything: ρ = 0.3 (independent behavior)
- Cross-sector: ρ = 0.3

**Random events:** ~0.1% chance per tick per ticker of a sudden 2–5% shock (up or down), adding realism/drama.

**`SimulatorDataSource`** wraps `GBMSimulator` in a `MarketDataSource` that runs a background asyncio task updating the `PriceCache` every 500ms.

### `MassiveDataSource` (massive_client.py)
Real market data via the Massive (Polygon.io) REST API:
- Polls `GET /v2/snapshot/locale/us/markets/stocks/tickers` for all watched tickers
- Default poll interval: 15 seconds (free tier: 5 req/min)
- Runs the synchronous Massive client in a thread (`asyncio.to_thread`) to avoid blocking the event loop
- Converts Massive millisecond timestamps to Unix seconds
- Gracefully skips malformed snapshots; logs but does not crash on API errors

### `create_market_data_source` (factory.py)
Environment-variable-driven factory:
- `MASSIVE_API_KEY` set and non-empty → `MassiveDataSource` (real data)
- Otherwise → `SimulatorDataSource` (default, no external deps)

### SSE Streaming (stream.py)
FastAPI router with a single endpoint: `GET /api/stream/prices`
- Media type: `text/event-stream`
- Emits all current ticker prices as JSON whenever the cache version changes
- Checks `request.is_disconnected()` to detect client drop
- Includes `retry: 1000` directive for automatic browser reconnect
- Headers disable nginx buffering for real-time delivery

**Usage:**
```python
from app.market import create_stream_router
router = create_stream_router(price_cache)
app.include_router(router)
```

---

## Public API

```python
from app.market import (
    PriceUpdate,              # Immutable price snapshot
    PriceCache,               # Thread-safe price store
    MarketDataSource,         # Abstract base class
    create_market_data_source, # Factory
    create_stream_router,     # SSE router factory
)
```

---

## Default Tickers and Seed Prices

| Ticker | Seed Price | σ (volatility) | μ (drift) |
|--------|-----------|-----------------|-----------|
| AAPL   | $190.00   | 22%             | 5%        |
| GOOGL  | $175.00   | 25%             | 5%        |
| MSFT   | $420.00   | 20%             | 5%        |
| AMZN   | $185.00   | 28%             | 5%        |
| TSLA   | $250.00   | 50%             | 3%        |
| NVDA   | $800.00   | 40%             | 8%        |
| META   | $500.00   | 30%             | 5%        |
| JPM    | $195.00   | 18%             | 4%        |
| V      | $280.00   | 17%             | 4%        |
| NFLX   | $600.00   | 35%             | 5%        |

Unknown tickers default to: σ=25%, μ=5%, random seed price $50–$300.

---

## Tests

Full unit and integration test suite in `backend/tests/market/`:

| File | What it tests |
|------|---------------|
| `test_models.py` | `PriceUpdate` dataclass, change/direction/serialization |
| `test_cache.py` | `PriceCache` CRUD, thread safety, version counter |
| `test_simulator.py` | `GBMSimulator` GBM math, correlation, add/remove tickers |
| `test_simulator_source.py` | `SimulatorDataSource` async lifecycle, cache integration |
| `test_massive.py` | `MassiveDataSource` with mocked API, error handling |
| `test_factory.py` | Factory selects correct implementation based on env vars |

Run tests:
```bash
cd backend
uv run --extra dev pytest -v
uv run --extra dev pytest --cov=app   # with coverage
```
