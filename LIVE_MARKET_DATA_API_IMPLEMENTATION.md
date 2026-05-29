# Live Market Data — API Integration Implementation Plan

> **Scope:** Backend / API integration **only**. Web and mobile UI integration are out of scope for this document and will follow in a separate plan once the API is verified.
>
> **Stage:** Replace mock market data with real, production-grade data sources across all 4 supported chains + Top-10 reference assets.
>
> **Client Decisions (confirmed on WhatsApp):**
>
> 1. **Binance API** — approved for spot/CEX market prices.
> 2. **Lithosphere chain** — use the **Ignite DEX API** (built in-house) for the only supported token: **LAX** ($1.0001 / LAX).
> 3. **Symbols** — show **Top 10 crypto** + **user-activated agent symbols** (union of both sets).
> 4. **Arbitrum / Base / BNB Chain** — use **GeckoTerminal** API (free tier, single integration covers all three).
>
> **References:**
>
> - `ProjectScope.txt` §4.3 (Web Live Market Data), §6.3 (Real Market Data backend)
> - `ClientApprovedDoc.txt` Web #3, Mobile #3
> - Existing infra: `services/quantt-backend/quantt/providers/`

---

## 1. Goals


| Goal                             | Outcome                                                                                  |
| -------------------------------- | ---------------------------------------------------------------------------------------- |
| Replace `DemoMarketDataProvider` | Real prices, real OHLCV, real volume                                                     |
| Single API surface               | One `MarketDataProvider` interface, multiple internal sources                            |
| Multi-source fan-out             | Binance + GeckoTerminal + Ignite DEX combined transparently                              |
| Real-time + REST hybrid          | Binance WebSocket for live ticks; REST for OHLCV/news/sentiment                          |
| Cache layer                      | Redis for repeat queries (rate-limit safety + speed)                                     |
| Provider isolation               | Each upstream API in its own module; fall back gracefully                                |
| AI Engine ready                  | Analysts (Fundamental / Sentiment / News / Technical) consume the new provider unchanged |


---

## 2. Current Codebase State (verified)


| File                                                          | What It Does                                                                                                                                  | Action                                       |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `services/quantt-backend/quantt/providers/base.py`            | Defines `MarketDataProvider` ABC with 5 methods (`get_snapshot`, `get_fundamentals`, `get_news`, `get_sentiment`, `get_technical_indicators`) | **Keep — do not change the interface**       |
| `services/quantt-backend/quantt/providers/demo_market.py`     | Returns deterministic fake data                                                                                                               | **Replace with real provider**               |
| `services/quantt-backend/quantt/providers/factory.py`         | `get_market_provider()` hard-codes `DemoMarketDataProvider()`                                                                                 | **Switch on env var**                        |
| `services/quantt-backend/quantt/providers/lithosphere_rpc.py` | JSON-RPC client for Lithosphere chain                                                                                                         | **Keep — extend with Ignite DEX REST calls** |
| `services/quantt-backend/quantt/providers/redis_cache.py`     | Redis cache helper (already present)                                                                                                          | **Reuse for caching market responses**       |
| `services/quantt-backend/quantt/settings.py`                  | App config                                                                                                                                    | **Add new env vars**                         |
| `services/quantt-backend/quantt/types.py`                     | `MarketSnapshot` model                                                                                                                        | **Extend if needed (e.g. add OHLCV field)**  |


**Why this matters:** The `MarketDataProvider` interface is already consumed by all 4 analysts via dependency injection from `factory.py`. We can swap the underlying implementation **without touching any analyst code**.

---

## 3. Target Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │   RealMarketDataProvider  (new)             │
                    │   implements MarketDataProvider             │
                    │                                             │
                    │   .get_snapshot(symbol, as_of)              │
                    │   .get_fundamentals(symbol, as_of)          │
                    │   .get_news(symbol, as_of)                  │
                    │   .get_sentiment(symbol, as_of)             │
                    │   .get_technical_indicators(symbol, as_of)  │
                    └────────────┬────────────────────────────────┘
                                 │ (routing logic)
       ┌─────────────────────────┼─────────────────────────────────────┐
       │                         │                                     │
       ▼                         ▼                                     ▼
┌─────────────────┐    ┌──────────────────────┐         ┌──────────────────────────┐
│ BinanceSource   │    │ GeckoTerminalSource  │         │ IgniteDexSource (LAX)    │
│ (CEX spot,      │    │ (DEX prices on       │         │ (Lithosphere chain only) │
│  Top-10 coins)  │    │  Arbitrum/Base/BNB)  │         │                          │
│ WS + REST       │    │ REST                 │         │ REST + RPC fallback      │
└─────────────────┘    └──────────────────────┘         └──────────────────────────┘
       │                         │                                     │
       └─────────────────────────┼─────────────────────────────────────┘
                                 │
                                 ▼
                       ┌──────────────────────┐
                       │  Redis cache layer    │  TTL: 5s prices / 60s OHLCV
                       │  (redis_cache.py)     │
                       └──────────────────────┘
                                 │
                                 ▼
                       ┌──────────────────────┐
                       │  News + Sentiment     │
                       │  CryptoPanic + GPT-4  │  (separate channel)
                       └──────────────────────┘
```

### Symbol Routing Rules


| Symbol type                                                                                | Source                                            |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `BTC`, `ETH`, `BNB`, `SOL`, `XRP`, `ADA`, `DOGE`, `TRX`, `AVAX`, `LINK` (Top-10 reference) | **Binance**                                       |
| Token on Arbitrum / Base / BNB DEX (e.g. user's agent trades `ARB/USDC` on Uniswap)        | **GeckoTerminal**                                 |
| `LAX` (Lithosphere)                                                                        | **Ignite DEX API**                                |
| Anything else                                                                              | Fallback to Binance lookup → if 404, return error |


The provider does this routing **internally** — analysts always just call `provider.get_snapshot("ETH", ...)` and the provider picks the right upstream.

---

## 4. File-by-File Implementation

### 4.1 New folder: `services/quantt-backend/quantt/providers/market/`

```
market/
├── __init__.py
├── provider.py          # RealMarketDataProvider (the orchestrator)
├── symbol_router.py     # Decides which source per symbol
├── sources/
│   ├── __init__.py
│   ├── binance.py       # Binance REST + WebSocket
│   ├── geckoterminal.py # GeckoTerminal REST
│   └── ignite_dex.py    # Ignite DEX REST (Lithosphere / LAX)
├── indicators.py        # MACD, RSI computation (pandas-ta)
├── news.py              # CryptoPanic adapter
└── sentiment.py         # GPT-4-based scoring
```

---

### 4.2 `market/sources/binance.py`

**Endpoints used:**

- `GET https://api.binance.com/api/v3/ticker/24hr?symbol=BTCUSDT` — 24h ticker
- `GET https://api.binance.com/api/v3/klines?symbol=BTCUSDT&interval=1h&limit=100` — OHLCV
- `wss://stream.binance.com:9443/ws/btcusdt@trade` — live trades (for future SSE push)

**Class skeleton:**

```python
class BinanceSource:
    BASE_REST = "https://api.binance.com"

    def __init__(self, http: httpx.AsyncClient, cache: RedisCache):
        self.http = http
        self.cache = cache

    async def ticker_24h(self, symbol: str) -> dict:
        key = f"binance:24h:{symbol}"
        cached = await self.cache.get(key)
        if cached: return cached
        r = await self.http.get(f"{self.BASE_REST}/api/v3/ticker/24hr",
                                params={"symbol": f"{symbol}USDT"})
        r.raise_for_status()
        data = r.json()
        await self.cache.set(key, data, ttl=5)
        return data

    async def klines(self, symbol: str, interval="1h", limit=100) -> list[list]:
        key = f"binance:klines:{symbol}:{interval}:{limit}"
        cached = await self.cache.get(key)
        if cached: return cached
        r = await self.http.get(f"{self.BASE_REST}/api/v3/klines",
                                params={"symbol": f"{symbol}USDT",
                                        "interval": interval, "limit": limit})
        r.raise_for_status()
        data = r.json()
        await self.cache.set(key, data, ttl=60)
        return data
```

**Top-10 symbol whitelist (constant):**

```python
TOP10 = ["BTC", "ETH", "BNB", "SOL", "XRP", "ADA", "DOGE", "TRX", "AVAX", "LINK"]
```

**Important Binance gotchas:**

- US users: Binance is geo-blocked → fallback to `api.binance.us` or Coinbase if `403`
- Rate limit: 1200 requests/minute per IP — Redis cache must absorb most traffic
- Symbol pair convention: Binance uses `BTCUSDT` (concatenated, no slash)

---

### 4.3 `market/sources/geckoterminal.py`

**Endpoints used:**

- `GET https://api.geckoterminal.com/api/v2/networks/{network}/tokens/{address}` — token price
- `GET https://api.geckoterminal.com/api/v2/networks/{network}/pools/{pool_address}/ohlcv/{timeframe}` — OHLCV

**Network ID mapping:**


| QUANTT Chain | GeckoTerminal `network` |
| ------------ | ----------------------- |
| Arbitrum     | `arbitrum`              |
| Base         | `base`                  |
| BNB Chain    | `bsc`                   |


**Class skeleton:**

```python
class GeckoTerminalSource:
    BASE = "https://api.geckoterminal.com/api/v2"

    NETWORK_MAP = {"arbitrum": "arbitrum", "base": "base", "bnb": "bsc"}

    async def token_price(self, chain: str, token_address: str) -> dict:
        net = self.NETWORK_MAP[chain]
        key = f"gecko:price:{net}:{token_address}"
        cached = await self.cache.get(key)
        if cached: return cached
        r = await self.http.get(f"{self.BASE}/networks/{net}/tokens/{token_address}")
        r.raise_for_status()
        data = r.json()["data"]["attributes"]
        await self.cache.set(key, data, ttl=10)
        return data

    async def ohlcv(self, chain: str, pool: str, timeframe="hour", limit=100) -> list:
        # timeframe: "minute" | "hour" | "day"
        net = self.NETWORK_MAP[chain]
        key = f"gecko:ohlcv:{net}:{pool}:{timeframe}:{limit}"
        cached = await self.cache.get(key)
        if cached: return cached
        r = await self.http.get(
            f"{self.BASE}/networks/{net}/pools/{pool}/ohlcv/{timeframe}",
            params={"limit": limit})
        r.raise_for_status()
        data = r.json()["data"]["attributes"]["ohlcv_list"]
        await self.cache.set(key, data, ttl=60)
        return data
```

**Gotchas:**

- Free tier: ~30 requests/minute — cache aggressively
- Needs **token contract address** OR **pool address** — not just a symbol name
- Build a small `agent_symbol → (chain, token_address, pool_address)` map in the DB (added when the user activates a symbol on an agent)

---

### 4.4 `market/sources/ignite_dex.py`

**Confirmed by client:** Ignite DEX API was built in-house by our team. It currently supports **1 token only: LAX = $1.0001**.

**Class skeleton (interface placeholder until we have exact endpoint):**

```python
class IgniteDexSource:
    """
    Lithosphere / Ignite DEX adapter.
    Currently supports LAX only — extend as more tokens list.
    """

    SUPPORTED = {"LAX"}

    def __init__(self, http: httpx.AsyncClient, cache: RedisCache, base_url: str):
        self.http = http
        self.cache = cache
        self.base = base_url  # e.g. https://api.ignite-dex.internal

    async def price(self, symbol: str) -> dict:
        if symbol not in self.SUPPORTED:
            raise ValueError(f"Ignite DEX does not list {symbol}")
        key = f"ignite:price:{symbol}"
        cached = await self.cache.get(key)
        if cached: return cached
        r = await self.http.get(f"{self.base}/v1/price/{symbol}")
        r.raise_for_status()
        data = r.json()
        # Expected shape: {"symbol": "LAX", "price": 1.0001, "ts": "..."}
        await self.cache.set(key, data, ttl=15)
        return data

    async def ohlcv(self, symbol: str, interval="1h", limit=100):
        # Stub — confirm with Ignite team if endpoint exists
        ...
```

**ACTION ITEMS:**

- Confirm with **@KaJ / Ignite DEX team** the exact endpoint paths and response shapes
- Confirm whether Ignite DEX exposes OHLCV history or only spot price
- Get the production base URL for the Ignite DEX API

---

### 4.5 `market/symbol_router.py`

```python
from .sources.binance import BinanceSource, TOP10
from .sources.geckoterminal import GeckoTerminalSource
from .sources.ignite_dex import IgniteDexSource

class SymbolRouter:
    """Picks the correct upstream source for each symbol."""

    def __init__(self, binance, gecko, ignite, symbol_registry):
        self.binance = binance
        self.gecko = gecko
        self.ignite = ignite
        # symbol_registry: DB-backed map of user-activated symbols
        #   { "ARB": {"chain": "arbitrum", "address": "0x...", "pool": "0x..."}, ... }
        self.registry = symbol_registry

    async def resolve(self, symbol: str):
        symbol = symbol.upper()
        if symbol in TOP10:
            return ("binance", self.binance)
        if symbol == "LAX":
            return ("ignite", self.ignite)
        entry = await self.registry.get(symbol)
        if entry:
            return ("gecko", self.gecko, entry)
        # Last-resort fallback
        return ("binance", self.binance)
```

---

### 4.6 `market/provider.py` — the orchestrator

```python
from quantt.providers.base import MarketDataProvider
from quantt.types import MarketSnapshot
from .indicators import compute_macd, compute_rsi
from .news import CryptoPanicNews
from .sentiment import GptSentimentScorer

class RealMarketDataProvider(MarketDataProvider):
    def __init__(self, router: SymbolRouter, news: CryptoPanicNews,
                 sentiment: GptSentimentScorer):
        self.router = router
        self.news = news
        self.sentiment = sentiment

    async def get_snapshot(self, symbol: str, as_of: str) -> MarketSnapshot:
        resolved = await self.router.resolve(symbol)
        if resolved[0] == "binance":
            t = await resolved[1].ticker_24h(symbol)
            return MarketSnapshot(
                symbol=symbol, as_of=as_of,
                price=float(t["lastPrice"]),
                change_pct_24h=float(t["priceChangePercent"]),
                volume_24h=float(t["quoteVolume"]),
                realized_volatility=None,  # compute from klines if needed
                metadata={"source": "binance"},
            )
        if resolved[0] == "gecko":
            _, gecko, entry = resolved
            t = await gecko.token_price(entry["chain"], entry["address"])
            return MarketSnapshot(
                symbol=symbol, as_of=as_of,
                price=float(t["price_usd"]),
                change_pct_24h=float(t.get("price_change_percentage", {}).get("h24", 0)),
                volume_24h=float(t.get("volume_usd", {}).get("h24", 0)),
                metadata={"source": "geckoterminal", "chain": entry["chain"]},
            )
        if resolved[0] == "ignite":
            t = await resolved[1].price(symbol)
            return MarketSnapshot(
                symbol=symbol, as_of=as_of,
                price=float(t["price"]),
                change_pct_24h=0.0,  # Ignite may not return 24h delta yet
                volume_24h=0.0,
                metadata={"source": "ignite_dex"},
            )
        raise RuntimeError(f"No source for symbol {symbol}")

    async def get_technical_indicators(self, symbol: str, as_of: str) -> dict:
        resolved = await self.router.resolve(symbol)
        if resolved[0] == "binance":
            klines = await resolved[1].klines(symbol, "1h", 100)
            closes = [float(k[4]) for k in klines]
        elif resolved[0] == "gecko":
            _, gecko, entry = resolved
            ohlcv = await gecko.ohlcv(entry["chain"], entry["pool"], "hour", 100)
            closes = [row[4] for row in ohlcv]
        else:
            return {"rsi": None, "macd_signal": None}  # Ignite OHLCV TBD

        return {
            "rsi": compute_rsi(closes, period=14),
            "macd_signal": compute_macd(closes),
        }

    async def get_news(self, symbol: str, as_of: str) -> list[dict]:
        return await self.news.fetch(symbol, limit=10)

    async def get_sentiment(self, symbol: str, as_of: str) -> dict:
        news = await self.news.fetch(symbol, limit=20)
        return await self.sentiment.score(symbol, news)

    async def get_fundamentals(self, symbol: str, as_of: str) -> dict:
        # Phase 2: hook to a fundamentals provider (Messari / CoinGecko)
        # For now: minimal static map for Top-10
        return FUNDAMENTALS_TOP10.get(symbol, {})
```

---

### 4.7 `market/indicators.py`

```python
import pandas as pd
import pandas_ta as ta

def compute_rsi(closes: list[float], period: int = 14) -> float | None:
    if len(closes) < period + 1:
        return None
    s = pd.Series(closes)
    rsi = ta.rsi(s, length=period)
    return float(rsi.iloc[-1]) if rsi is not None else None

def compute_macd(closes: list[float]) -> dict | None:
    if len(closes) < 35:
        return None
    s = pd.Series(closes)
    macd = ta.macd(s, fast=12, slow=26, signal=9)
    if macd is None or macd.empty:
        return None
    return {
        "macd": float(macd.iloc[-1, 0]),
        "histogram": float(macd.iloc[-1, 1]),
        "signal": float(macd.iloc[-1, 2]),
        "verdict": "bullish" if macd.iloc[-1, 1] > 0 else "bearish",
    }
```

Add to `pyproject.toml`:

```toml
pandas-ta = "^0.3.14b0"
pandas = "^2.2.0"
```

---

### 4.8 `market/news.py`

```python
class CryptoPanicNews:
    BASE = "https://cryptopanic.com/api/v1/posts/"

    def __init__(self, http, cache, api_key: str):
        self.http = http
        self.cache = cache
        self.key = api_key

    async def fetch(self, symbol: str, limit: int = 10) -> list[dict]:
        cache_key = f"news:{symbol}:{limit}"
        cached = await self.cache.get(cache_key)
        if cached: return cached
        r = await self.http.get(self.BASE, params={
            "auth_token": self.key,
            "currencies": symbol,
            "public": "true",
        })
        r.raise_for_status()
        results = r.json().get("results", [])[:limit]
        items = [{
            "headline": x["title"],
            "url": x["url"],
            "source": x["source"]["domain"],
            "published_at": x["published_at"],
            "impact": _classify(x.get("votes", {})),
        } for x in results]
        await self.cache.set(cache_key, items, ttl=300)  # 5 min
        return items
```

---

### 4.9 `market/sentiment.py`

```python
class GptSentimentScorer:
    """Use existing OpenAI provider to score news headlines into sentiment."""

    def __init__(self, llm_provider):
        self.llm = llm_provider

    async def score(self, symbol: str, news: list[dict]) -> dict:
        if not news:
            return {"score": 0.5, "trend": "neutral", "sources": []}

        headlines = "\n".join(f"- {n['headline']}" for n in news[:20])
        result = await self.llm.complete_structured(
            system_prompt="You score crypto sentiment 0.0 (very bearish) to 1.0 (very bullish).",
            user_prompt=f"Symbol: {symbol}\nHeadlines:\n{headlines}\nReturn JSON: {{score, trend, summary}}",
            schema=SentimentSchema,
        )
        return {
            "score": result.score,
            "trend": result.trend,
            "sources": list({n["source"] for n in news}),
            "summary": result.summary,
        }
```

---

### 4.10 Update `providers/factory.py`

```python
def get_market_provider():
    if settings.market_provider == "demo":
        return DemoMarketDataProvider()

    http = httpx.AsyncClient(timeout=10.0)
    cache = RedisCache.from_settings()
    binance = BinanceSource(http, cache)
    gecko = GeckoTerminalSource(http, cache)
    ignite = IgniteDexSource(http, cache, settings.ignite_dex_url)
    registry = SymbolRegistry()  # DB-backed user-activated symbols
    router = SymbolRouter(binance, gecko, ignite, registry)
    news = CryptoPanicNews(http, cache, settings.cryptopanic_key)
    sentiment = GptSentimentScorer(get_llm_provider())

    return RealMarketDataProvider(router, news, sentiment)
```

---

### 4.11 Update `settings.py`

Add:

```python
class Settings(BaseSettings):
    # … existing fields
    market_provider: str = "demo"                   # "demo" | "real"
    binance_base_url: str = "https://api.binance.com"
    geckoterminal_base_url: str = "https://api.geckoterminal.com/api/v2"
    ignite_dex_url: str = "https://api.ignite-dex.internal"  # TBD with team
    cryptopanic_key: str = ""
```

`.env` additions:

```bash
MARKET_PROVIDER=real
BINANCE_BASE_URL=https://api.binance.com
GECKOTERMINAL_BASE_URL=https://api.geckoterminal.com/api/v2
IGNITE_DEX_URL=<confirm with Ignite team>
CRYPTOPANIC_KEY=<get free key from cryptopanic.com>
```

---

## 5. Database: Symbol Registry

To route DEX symbols (Arbitrum / Base / BNB) we need a mapping table populated when users activate a token on an agent.

### Prisma schema addition (`apps/api/prisma/schema.prisma`)

```prisma
model SymbolRegistry {
  id           String   @id @default(cuid())
  symbol       String   @unique           // "ARB", "AERO", "CAKE", "LAX", "ETH"
  source       String                     // "binance" | "geckoterminal" | "ignite_dex"
  chain        String?                    // "arbitrum" | "base" | "bnb" | "lithosphere"
  tokenAddress String?                    // EVM address (lowercased)
  poolAddress  String?                    // Primary trading pool for OHLCV
  decimals     Int?     @default(18)
  activatedAt  DateTime @default(now())
  agentIds     String[]                   // Which agents reference this symbol
}
```

Seed the Top-10 + LAX at migration time. New symbols are added by the agent-creation API endpoint when a user picks a chain + token.

---

## 6. API Endpoints (Node Gateway — `apps/api`)

These are the **thin REST endpoints** the web/mobile clients will call. They proxy to the Python AI engine, which uses the market provider above.

### 6.1 `GET /v1/market/snapshot?symbol=ETH`

Returns current price + 24h change + volume.

**Response:**

```json
{
  "symbol": "ETH",
  "price": 3812.55,
  "change_pct_24h": 1.84,
  "volume_24h": 18234982341.22,
  "source": "binance",
  "as_of": "2026-05-21T10:34:51Z"
}
```

### 6.2 `GET /v1/market/ohlcv?symbol=ETH&interval=1h&limit=100`

Returns candlestick array.

**Response:**

```json
{
  "symbol": "ETH",
  "interval": "1h",
  "candles": [
    { "t": 1716287400000, "o": 3805.1, "h": 3815.5, "l": 3801.0, "c": 3812.5, "v": 1284.5 },
    ...
  ],
  "source": "binance"
}
```

### 6.3 `GET /v1/market/indicators?symbol=ETH`

Returns MACD + RSI (Technical Analyst consumes this too).

### 6.4 `GET /v1/market/news?symbol=ETH&limit=10`

CryptoPanic-backed news list.

### 6.5 `GET /v1/market/sentiment?symbol=ETH`

GPT-4 sentiment score over recent news.

### 6.6 `GET /v1/market/top10`

Convenience endpoint — returns snapshot for all Top-10 in one call (caches aggressively).

### 6.7 `GET /v1/market/watchlist`  *(authenticated)*

Returns snapshots for **Top-10 + user's agent-activated symbols** in one call. This is what the Dashboard Live Market Data widget calls.

### 6.8 *(Phase 2)* `GET /v1/market/stream` — SSE for live ticks

Forwards Binance WebSocket trade events through the existing telemetry SSE channel, scoped by symbol query param.

---

## 7. Caching Strategy


| Data                         | TTL    | Reason                                             |
| ---------------------------- | ------ | -------------------------------------------------- |
| Spot price (24h ticker)      | 5 sec  | Fresh enough for dashboards, slashes Binance load  |
| OHLCV candles                | 60 sec | Bars only close on the minute/hour boundary anyway |
| Indicators (MACD/RSI)        | 60 sec | Tied to OHLCV cache                                |
| News                         | 5 min  | News changes slowly                                |
| Sentiment (LLM-scored)       | 15 min | LLM calls are expensive                            |
| Top-10 watchlist (composite) | 5 sec  | Composed of cached parts                           |


All caches use the existing `redis_cache.py` helper. Cache key prefix: `market:<source>:<symbol>:<...>`.

---

## 8. Error Handling & Fallback Matrix


| Failure                          | Behavior                                                                            |
| -------------------------------- | ----------------------------------------------------------------------------------- |
| Binance returns 403 (geo-block)  | Try `api.binance.us` → if still failing, return stale cache with `stale: true` flag |
| GeckoTerminal rate-limited (429) | Return stale cache or 503 with `Retry-After` header                                 |
| Ignite DEX 5xx                   | Fall back to last known price + warning event on telemetry                          |
| Symbol not found in any source   | 404 with explicit `{ "error": "symbol_unknown", "symbol": "XYZ" }`                  |
| LLM (sentiment) timeout          | Return `{ score: 0.5, trend: "unknown" }` — never block the snapshot                |
| Redis down                       | Bypass cache, call upstream directly (degraded mode)                                |


Every fallback also publishes a telemetry event (`market.source.degraded`) so ops can monitor.

---

## 9. Testing Plan

### 9.1 Unit tests (`tests/providers/market/`)

- `test_binance_ticker.py` — mock `httpx`, verify parsing
- `test_geckoterminal_token.py` — mock response, verify chain mapping
- `test_ignite_lax.py` — mock the Ignite API, verify LAX-only acceptance
- `test_symbol_router.py` — Top-10 routes to Binance, LAX to Ignite, custom → Gecko
- `test_indicators.py` — RSI & MACD against fixed input series with known values

### 9.2 Integration tests

- `test_real_provider_integration.py` — hits real Binance/Gecko with the **public-only Top-10 fixtures**, marked `@pytest.mark.integration` (skipped in CI by default, runs in nightly)

### 9.3 Manual verification commands

```bash
# In the Python service
python -m quantt market snapshot --symbol BTC
python -m quantt market snapshot --symbol LAX
python -m quantt market snapshot --symbol ARB        # via Gecko
python -m quantt market indicators --symbol ETH
python -m quantt market news --symbol BTC
```

---

## 10. Phased Rollout


| Phase               | Includes                                     | Days        | Status                              |
| ------------------- | -------------------------------------------- | ----------- | ----------------------------------- |
| **P1**              | Binance source + Top-10 + Redis caching      | 1.5         | ⏳ Ready to start                    |
| **P2**              | Symbol registry table + GeckoTerminal source | 1.5         | ⏳ Ready (needs DB migration)        |
| **P3**              | Ignite DEX integration (LAX)                 | 0.5         | ⏸ Confirm endpoints with team first |
| **P4**              | Indicators (MACD / RSI)                      | 0.5         | ⏳                                   |
| **P5**              | News (CryptoPanic) + Sentiment (GPT-4)       | 1           | ⏳ Needs `CRYPTOPANIC_KEY`           |
| **P6**              | REST endpoints in `apps/api`                 | 1           | ⏳                                   |
| **P7**              | Tests + error/fallback hardening             | 1           | ⏳                                   |
| **P8** *(optional)* | WebSocket → SSE live price stream            | 1.5         | 🔜 Phase 2                          |
| **Total (P1–P7)**   |                                              | **~7 days** |                                     |


---

## 11. Open Questions to Resolve Before Starting

1. **Ignite DEX endpoint paths** — confirm with @KaJ / Ignite team. Specifically:
  - GET price URL
  - Does it expose OHLCV history?
  - Auth required? API key?
2. **CryptoPanic free tier** — confirm 200 req/day limit is enough for our user base, else upgrade plan.
3. **Symbol registry seed list** — confirm exact Top-10 set with client. Document above uses CoinMarketCap's Top-10 (excluding stablecoins).
4. **Volume figures unit** — Binance returns `quoteVolume` in USDT, GeckoTerminal in USD, Ignite TBD. Normalize all to **USD** in `MarketSnapshot`.
5. **Stablecoins** — should USDT / USDC / DAI also be shown in the watchlist, or excluded?
6. **Historical depth** — for OHLCV, 100 candles enough or do we need 500+ for backtesting?

---

## 12. What to Do Next (Action Sequence)

1. ✅ **Client questions answered** — sources confirmed.
2. ⏳ **Confirm Ignite DEX API spec with internal team** — block on this for P3 only; P1, P2, P4-P7 can proceed.
3. ⏳ **Get CryptoPanic free API key** — sign up at [https://cryptopanic.com/developers/api/](https://cryptopanic.com/developers/api/).
4. ⏳ **Create folder structure** — `services/quantt-backend/quantt/providers/market/`.
5. ⏳ **Implement P1** — Binance + cache + Top-10 routing → green tests.
6. ⏳ **Run Prisma migration** — add `SymbolRegistry` table, seed Top-10 + LAX.
7. ⏳ **Implement P2 → P7** in sequence as listed.
8. ⏳ **Switch env** `MARKET_PROVIDER=real` in staging → smoke test → promote to prod.
9. 🔜 **Hand off to web/mobile teams** for UI consumption once `/v1/market/*` endpoints are live.

---

## 13. Notes on the Existing Code

- **Do not delete `DemoMarketDataProvider`** — keep it as a test fixture (`MARKET_PROVIDER=demo` in unit tests).
- **The `MarketDataProvider` interface in `base.py` is the contract** — analysts depend on it. Do not break method signatures; only add new optional methods if absolutely needed.
- **Reuse `redis_cache.py`** — do not introduce a second cache library.
- **Reuse `LithosphereRpcClient`** — Ignite DEX wraps over the same chain; if Ignite REST is unavailable, we can fall back to raw RPC reads of the LAX pool contract.

---

## 14. Done When

- `GET /v1/market/snapshot?symbol=BTC` returns real Binance price
- `GET /v1/market/snapshot?symbol=LAX` returns `$1.0001` from Ignite DEX
- `GET /v1/market/snapshot?symbol=ARB` returns real Arbitrum DEX price via GeckoTerminal
- `GET /v1/market/watchlist` returns Top-10 + user's agent symbols in one payload
- All 4 analysts (`fundamental`, `sentiment`, `news`, `technical`) read from `RealMarketDataProvider` with zero code changes inside the analysts themselves
- Redis cache hit rate > 80% under simulated dashboard load (`siege` or `k6`)
- Every external API failure produces a graceful response + telemetry warning event
- Unit tests pass; integration tests pass in nightly CI

