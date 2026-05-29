# Live Market Data — Web Integration Plan

> **Scope:** Web app (`apps/web`) integration of the Live Market Data API endpoints built in [LIVE_MARKET_DATA_API_IMPLEMENTATION.md](LIVE_MARKET_DATA_API_IMPLEMENTATION.md).
>
> **Prerequisite:** All `/v1/market/*` endpoints must be live on the API gateway before starting this work.
>
> **Out of scope:** Mobile integration (separate doc), API/backend implementation (already documented).
>
> **References:**
> - `ProjectScope.txt` §4.2 Dashboard, §4.3 Live Market Data, §4.5 Agent Detail
> - `ClientApprovedDoc.txt` Web #2 + #3
> - Existing app: `apps/web/app/market/`, `apps/web/app/dashboard/`, `apps/web/app/agents/`

---

## 1. Goals

| Goal | Outcome |
|------|---------|
| Replace `MARKET_DATA` mock | Real prices on every page |
| Wire watchlist (Top-10 + agent symbols) | Personalised market view per user |
| Add OHLCV chart component | Reusable candlestick + line chart |
| Add MACD / RSI indicator panels | Show technical signals on Agent Detail |
| News + sentiment widgets | On Dashboard + per-symbol drill-down |
| Live updates | Polling (Phase 1) → SSE stream (Phase 2) |
| Strong typing | Shared types between API + web via `packages/types` |
| Loading + error UX | Skeletons, error boundaries, retry — no white screens |

---

## 2. Current Codebase State (verified)

| File | Current State | Action |
|------|---------------|--------|
| `apps/web/app/market/page.tsx` | Client component using `MARKET_DATA` mock | **Convert to server component → fetch real watchlist** |
| `apps/web/app/dashboard/page.tsx` | Dashboard shell | **Add LiveMarketWidget** |
| `apps/web/app/agents/[id]/page.tsx` | Agent Detail | **Add price chart + indicators for agent's traded symbol** |
| `apps/web/lib/client.ts` | Wraps `QuanttsApiClient`, attaches cookie auth | **Reuse — add market methods to client** |
| `apps/web/lib/mock-data.ts` | Contains `MARKET_DATA` and similar | **Keep for tests only; remove imports from pages** |
| `packages/api-client/src/index.ts` | Single-file API client | **Add typed `market.*` methods** |
| `apps/web/app/live/page.tsx` | Live page exists | **Repurpose — render telemetry + market events (later)** |
| `apps/web/components/dashboard/` | Has `activity-feed.tsx`, `metric-card.tsx`, `agent-table.tsx` | **Add `live-market-widget.tsx`, `price-chart.tsx`, `indicator-panel.tsx`** |

---

## 3. Target Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  NEXT.JS WEB APP (apps/web)                                  │
│                                                              │
│  Server Components (initial render, SEO, fast paint)         │
│   ├─ /dashboard          (LiveMarketWidget initial data)     │
│   ├─ /market             (full watchlist + filters)          │
│   └─ /agents/[id]        (per-agent symbol chart)            │
│                                                              │
│  Client Components (interactivity, polling, charts)          │
│   ├─ <PriceChart>        Recharts/lightweight-charts         │
│   ├─ <IndicatorPanel>    MACD / RSI bars                     │
│   ├─ <NewsList>          Auto-refresh CryptoPanic items      │
│   ├─ <SentimentBadge>    GPT-4 score → color pill            │
│   └─ <WatchlistTable>    Sortable, filterable                │
│                                                              │
│  Hooks                                                       │
│   ├─ useMarketSnapshot(symbol)     // 5-sec poll             │
│   ├─ useOhlcv(symbol, interval)    // 60-sec poll            │
│   ├─ useIndicators(symbol)         // 60-sec poll            │
│   ├─ useNews(symbol)               // 5-min poll             │
│   ├─ useWatchlist()                // composite              │
│   └─ useMarketStream(symbol)       // SSE (Phase 2)          │
│                                                              │
│  Shared client                                               │
│   └─ apps/web/lib/client.ts → @quantts/api-client.market.*   │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP / SSE
                  ┌───────────────────────┐
                  │  API Gateway          │
                  │  /v1/market/snapshot  │
                  │  /v1/market/ohlcv     │
                  │  /v1/market/watchlist │
                  │  /v1/market/news      │
                  │  /v1/market/stream    │  (Phase 2)
                  └───────────────────────┘
```

---

## 4. Data Flow Patterns

### Pattern A — Server-rendered initial data
Used for the first paint of `/market`, `/dashboard`, `/agents/[id]`.

```tsx
// app/market/page.tsx (server component)
import { getClient } from "@/lib/client";
import { WatchlistTable } from "@/components/market/watchlist-table";

export default async function MarketPage() {
  const api = await getClient();
  const initial = await api.market.watchlist();   // Server fetch (no flash)
  return <WatchlistTable initial={initial} />;
}
```

### Pattern B — Client-side polling
Used after first paint, by client components.

```tsx
// components/market/watchlist-table.tsx ("use client")
"use client";
import { useWatchlist } from "@/hooks/useWatchlist";

export function WatchlistTable({ initial }) {
  const { data } = useWatchlist({ initial, refetchInterval: 5_000 });
  return <table>…</table>;
}
```

### Pattern C — Live stream (Phase 2)
SSE bridges the latency gap once Phase 1 is shipped.

```tsx
const { lastTick } = useMarketStream(symbol);
```

---

## 5. File-by-File Plan

### 5.1 Extend the shared API client

**File:** `packages/api-client/src/index.ts`

Add a typed `market` namespace on `QuanttsApiClient`:

```typescript
export interface MarketSnapshot {
  symbol: string;
  price: number;
  change_pct_24h: number;
  volume_24h: number;
  source: "binance" | "geckoterminal" | "ignite_dex";
  as_of: string;
  stale?: boolean;
}

export interface OhlcvCandle {
  t: number; o: number; h: number; l: number; c: number; v: number;
}

export interface NewsItem {
  headline: string; url: string; source: string;
  published_at: string; impact: "positive" | "negative" | "neutral";
}

export interface Indicators {
  rsi: number | null;
  macd_signal: { macd: number; histogram: number; signal: number;
                 verdict: "bullish" | "bearish" } | null;
}

export interface Sentiment {
  score: number; trend: string; sources: string[]; summary?: string;
}

class MarketApi {
  constructor(private base: string, private token?: string) {}

  private async req<T>(path: string): Promise<T> {
    const r = await fetch(`${this.base}${path}`, {
      headers: this.token ? { Authorization: `Bearer ${this.token}` } : {},
      cache: "no-store",
    });
    if (!r.ok) throw new MarketError(r.status, await r.text());
    return r.json() as Promise<T>;
  }

  snapshot(symbol: string)        { return this.req<MarketSnapshot>(`/v1/market/snapshot?symbol=${symbol}`); }
  ohlcv(symbol: string, interval="1h", limit=100) {
                                    return this.req<{candles: OhlcvCandle[]}>(`/v1/market/ohlcv?symbol=${symbol}&interval=${interval}&limit=${limit}`); }
  indicators(symbol: string)      { return this.req<Indicators>(`/v1/market/indicators?symbol=${symbol}`); }
  news(symbol: string, limit=10)  { return this.req<NewsItem[]>(`/v1/market/news?symbol=${symbol}&limit=${limit}`); }
  sentiment(symbol: string)       { return this.req<Sentiment>(`/v1/market/sentiment?symbol=${symbol}`); }
  top10()                         { return this.req<MarketSnapshot[]>(`/v1/market/top10`); }
  watchlist()                     { return this.req<MarketSnapshot[]>(`/v1/market/watchlist`); }
}

// Attach to QuanttsApiClient
export class QuanttsApiClient {
  market: MarketApi;
  constructor(base: string, token?: string) {
    this.market = new MarketApi(base, token);
    // … existing namespaces
  }
}

export class MarketError extends Error {
  constructor(public status: number, public body: string) { super(`Market API ${status}: ${body}`); }
}
```

---

### 5.2 New hooks folder: `apps/web/hooks/market/`

Why a folder: keeps each hook small and individually testable.

```
hooks/market/
├── useMarketSnapshot.ts
├── useOhlcv.ts
├── useIndicators.ts
├── useNews.ts
├── useSentiment.ts
├── useWatchlist.ts
└── useMarketStream.ts          # Phase 2 only
```

**Example — `useWatchlist.ts`:**

```typescript
"use client";
import { useEffect, useState, useCallback } from "react";
import type { MarketSnapshot } from "@quantts/api-client";

export function useWatchlist(opts: {
  initial?: MarketSnapshot[];
  refetchInterval?: number;
}) {
  const [data, setData] = useState(opts.initial ?? []);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(!opts.initial);

  const fetchOnce = useCallback(async () => {
    try {
      const r = await fetch("/api/market/watchlist", { cache: "no-store" });
      if (!r.ok) throw new Error(String(r.status));
      setData(await r.json());
      setError(null);
    } catch (e) {
      setError(e as Error);
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    if (!opts.initial) void fetchOnce();
    if (opts.refetchInterval) {
      const id = setInterval(fetchOnce, opts.refetchInterval);
      return () => clearInterval(id);
    }
  }, [fetchOnce, opts.initial, opts.refetchInterval]);

  return { data, error, loading, refetch: fetchOnce };
}
```

**Notes:**
- The hook talks to `/api/market/watchlist` (a Next.js route handler that proxies the gateway with cookie auth — see §5.3).
- Polling cadence is configurable per hook caller.
- All hooks expose `{ data, error, loading, refetch }` — consistent shape.

---

### 5.3 Next.js Route Handlers (BFF proxy layer)

**Why:** the browser cannot read the `quantts_access` httpOnly cookie directly to hit the gateway. We need thin route handlers in `apps/web/app/api/market/*` that **read the cookie server-side and forward to the gateway**.

**File:** `apps/web/app/api/market/snapshot/route.ts`
```typescript
import { NextRequest, NextResponse } from "next/server";
import { getClient } from "@/lib/client";

export async function GET(req: NextRequest) {
  const symbol = req.nextUrl.searchParams.get("symbol");
  if (!symbol) return NextResponse.json({ error: "symbol required" }, { status: 400 });
  const api = await getClient();
  try {
    const data = await api.market.snapshot(symbol);
    return NextResponse.json(data);
  } catch (e: any) {
    return NextResponse.json({ error: e.message }, { status: 502 });
  }
}
```

Repeat the same thin handler pattern for:
- `/api/market/watchlist/route.ts`
- `/api/market/ohlcv/route.ts`
- `/api/market/indicators/route.ts`
- `/api/market/news/route.ts`
- `/api/market/sentiment/route.ts`
- `/api/market/top10/route.ts`

These are ~5 lines each.

---

### 5.4 New components

#### 5.4.1 `components/market/watchlist-table.tsx`

Replaces the mock `MARKET_DATA` table at `app/market/page.tsx`.

**Features:**
- Sortable columns: price, 24h change, volume
- Filter chips: All / Top-10 / My Agents / by chain (Arbitrum / Base / BNB / Lithosphere)
- Click row → navigate to `/market/[symbol]` (drill-down page)
- Stale-data indicator (orange dot) if `snapshot.stale === true`
- Empty state if user has zero agents activated

#### 5.4.2 `components/market/price-chart.tsx`

Candlestick + line dual mode chart.

**Library choice:** `lightweight-charts` (TradingView, MIT licensed, ~35 KB) — *not* Recharts (no candle support).

```bash
npm install --workspace=apps/web lightweight-charts
```

```tsx
"use client";
import { createChart, CandlestickSeries } from "lightweight-charts";
import { useEffect, useRef } from "react";
import { useOhlcv } from "@/hooks/market/useOhlcv";

export function PriceChart({ symbol, interval = "1h" }: {symbol: string, interval?: string}) {
  const ref = useRef<HTMLDivElement>(null);
  const { data } = useOhlcv(symbol, interval, 100);

  useEffect(() => {
    if (!ref.current || !data?.candles) return;
    const chart = createChart(ref.current, { width: ref.current.clientWidth, height: 320 });
    const series = chart.addSeries(CandlestickSeries);
    series.setData(data.candles.map(c => ({
      time: (c.t / 1000) as any, open: c.o, high: c.h, low: c.l, close: c.c,
    })));
    return () => chart.remove();
  }, [data]);

  return <div ref={ref} />;
}
```

#### 5.4.3 `components/market/indicator-panel.tsx`

```tsx
"use client";
import { useIndicators } from "@/hooks/market/useIndicators";

export function IndicatorPanel({ symbol }: { symbol: string }) {
  const { data, loading } = useIndicators(symbol);
  if (loading || !data) return <Skeleton />;
  return (
    <div className="indicator-grid">
      <IndicatorBox label="RSI (14)" value={data.rsi?.toFixed(1)} tone={rsiTone(data.rsi)} />
      <IndicatorBox label="MACD" value={data.macd_signal?.verdict ?? "—"} tone={data.macd_signal?.verdict} />
    </div>
  );
}
```

#### 5.4.4 `components/market/news-list.tsx`

Polls every 5 min, color-codes by `impact`, click headline → open `url` in new tab.

#### 5.4.5 `components/market/sentiment-badge.tsx`

Single color pill: red (< 0.4) / yellow (0.4–0.6) / green (> 0.6).

#### 5.4.6 `components/market/live-market-widget.tsx`

Dashboard widget — compact Top-5 view + "See all →" link to `/market`.

---

### 5.5 Page-by-page wiring

#### 5.5.1 `app/market/page.tsx`  → server component, full watchlist

```tsx
import { getClient } from "@/lib/client";
import { WatchlistTable } from "@/components/market/watchlist-table";
import { MarketSummaryStrip } from "@/components/market/summary-strip";

export const dynamic = "force-dynamic";

export default async function MarketPage() {
  const api = await getClient();
  const [watchlist, top10] = await Promise.all([
    api.market.watchlist().catch(() => []),
    api.market.top10().catch(() => []),
  ]);
  return (
    <div className="market-page">
      <MarketSummaryStrip data={top10} />
      <WatchlistTable initial={watchlist} />
    </div>
  );
}
```

**Remove** the import of `MARKET_DATA` from `lib/mock-data.ts`.

#### 5.5.2 `app/market/[symbol]/page.tsx`  → new drill-down page

Shows: PriceChart + IndicatorPanel + NewsList + SentimentBadge for one symbol.

```tsx
export default async function SymbolPage({ params }: { params: { symbol: string } }) {
  const api = await getClient();
  const symbol = params.symbol.toUpperCase();
  const [snap, news, sentiment] = await Promise.all([
    api.market.snapshot(symbol),
    api.market.news(symbol),
    api.market.sentiment(symbol),
  ]);
  return (
    <div>
      <SymbolHeader snapshot={snap} sentiment={sentiment} />
      <PriceChart symbol={symbol} />
      <IndicatorPanel symbol={symbol} />
      <NewsList initial={news} symbol={symbol} />
    </div>
  );
}
```

#### 5.5.3 `app/dashboard/page.tsx`  → add `LiveMarketWidget`

Add a panel under the existing metric cards showing **Top 5 watchlist tickers** with live price/change.

#### 5.5.4 `app/agents/[id]/page.tsx`  → per-agent symbol view

For each symbol the agent trades:
- Mini `PriceChart` (line mode, 24h)
- `IndicatorPanel` for the agent's primary symbol
- `SentimentBadge`

This wires Live Market Data **into the Agent Detail page** per `ClientApprovedDoc.txt` Web #3.

---

### 5.6 Loading and error states

| State | Component |
|-------|-----------|
| Initial load | `<Skeleton variant="row" rows={10} />` |
| Refetch (background) | Show small spinner in header, do **not** unmount table |
| API 502 | `<MarketUnavailable retry={refetch} />` |
| Symbol 404 | `<UnknownSymbol symbol={symbol} />` |
| Stale flag | Orange dot tooltip "Showing cached data, upstream slow" |

Each component must handle these — no white-screen-of-death.

---

### 5.7 Type sharing

Use `@quantts/api-client` exported types **everywhere** — never redeclare `MarketSnapshot` etc. in pages/components.

```typescript
import type { MarketSnapshot, OhlcvCandle, NewsItem } from "@quantts/api-client";
```

This keeps web + (later) mobile aligned.

---

## 6. UX Decisions

### 6.1 Refresh cadence (default)
| Surface | Interval | Why |
|---------|----------|-----|
| Dashboard widget | 10 sec | Background scan, not focus |
| `/market` watchlist | 5 sec | Active scanning surface |
| `/market/[symbol]` chart | 60 sec (auto) | Bars close on minute boundary |
| News list | 5 min | Slow-moving |
| Sentiment | 15 min | LLM-scored, expensive |

User can override via a "Pause updates" toggle in the topbar (saved to localStorage).

### 6.2 Stale indicator
When `snapshot.stale === true`, show an **orange dot** next to the row + tooltip explaining upstream is degraded.

### 6.3 Empty states
- New user with zero agents → watchlist shows only Top-10 + a CTA "Create an agent to track more symbols".
- Symbol unknown → "We don't have data for XYZ. Supported sources: Binance, GeckoTerminal, Ignite DEX."

### 6.4 Accessibility
- All color-coded badges must include a text label (don't rely on color alone).
- Charts must have an "Export data as CSV" link for screen-reader users.

---

## 7. Phased Rollout

| Phase | Includes | Days | Status |
|-------|----------|------|--------|
| **W1** | API client `market.*` namespace + Next.js route handlers | 0.5 | ⏳ Ready |
| **W2** | Hooks (`useMarketSnapshot`, `useOhlcv`, `useWatchlist`, etc.) | 1 | ⏳ |
| **W3** | `/market` page → real data, remove mock | 0.5 | ⏳ |
| **W4** | `PriceChart` + `IndicatorPanel` (lightweight-charts) | 1.5 | ⏳ |
| **W5** | `/market/[symbol]` drill-down page | 1 | ⏳ |
| **W6** | Dashboard `LiveMarketWidget` | 0.5 | ⏳ |
| **W7** | Agent Detail integration | 0.5 | ⏳ |
| **W8** | News + Sentiment components | 1 | ⏳ |
| **W9** | Loading / error / stale UX polish | 0.5 | ⏳ |
| **W10** | E2E tests (Playwright) | 1 | ⏳ |
| **W11** *(later)* | SSE live tick stream (Phase 2) | 1 | 🔜 |
| **Total (W1–W10)** | | **~7 days** | |

---

## 8. Testing

### 8.1 Unit (Vitest + React Testing Library)
- `useWatchlist` — mock fetch, verify polling, verify error path
- `WatchlistTable` — sort/filter behavior, click navigation
- `PriceChart` — chart receives candle data; cleanup on unmount

### 8.2 Integration
- Route handler tests — verify cookie is forwarded, errors map to 502/4xx

### 8.3 E2E (Playwright — already used in repo)
- Anonymous user → `/market` redirects to login
- Logged-in user → `/market` shows watchlist
- Agent Detail → indicator panel renders for agent's symbol
- Network failure → error component shows + retry works

### 8.4 Manual
```
1. Log in. Open /dashboard. Confirm LiveMarketWidget shows real BTC/ETH/LAX.
2. Open /market. Sort by 24h change. Filter to "My Agents".
3. Click a row → /market/ETH. Confirm chart, MACD, RSI, news, sentiment.
4. Disable network. Confirm error state appears. Re-enable → retry succeeds.
5. Kill the gateway. Confirm "Market unavailable" component (not crash).
```

---

## 9. Performance Considerations

- **`cache: "no-store"`** on all market fetches — prices must never be Next.js-cached at the data layer.
- **Suspense boundaries** wrap chart components so heavy `lightweight-charts` doesn't block first paint.
- **`dynamic = "force-dynamic"`** on server pages with market data.
- **Polling pauses** when tab is hidden (`document.visibilityState !== "visible"`).
- **Single watchlist call** powers Dashboard widget + `/market` page (request dedup via React Query — optional add).

> 🟡 **Optional dependency upgrade:** consider adding `@tanstack/react-query` for cache + dedup + retry. It would eliminate ~50% of the hook code. Decision: stick with hand-rolled hooks for Phase 1 to keep bundle small; revisit if hook count exceeds 10.

---

## 10. Open Questions

1. **Drill-down route shape** — `/market/[symbol]` vs `/market?symbol=ETH`? (Recommend route param for SEO + shareable URLs.)
2. **Currency display** — always USD, or honor a user setting?
3. **Locale / number formatting** — use `Intl.NumberFormat`? Confirm with design.
4. **Should the watchlist be reorderable / pinnable per user?** (Saves state to DB — adds scope.)
5. **Sentiment summary** — show full GPT summary text or only the badge?
6. **Should the Agent Detail page show prices for ALL the agent's chains, or only its primary chain?**

---

## 11. Acceptance Criteria

- [ ] `/market` page renders real watchlist (Top-10 + user's agent symbols) — no mock imports
- [ ] Sorting and chain-filter work
- [ ] Click row → `/market/[symbol]` shows live chart, MACD, RSI, news, sentiment
- [ ] Dashboard `LiveMarketWidget` updates without page reload
- [ ] Agent Detail shows price + indicators for the agent's primary symbol
- [ ] All loading states use `<Skeleton>`, never a flash of empty UI
- [ ] API gateway down → "Market unavailable" component, no crash
- [ ] Stale data flag visually surfaced
- [ ] Polling pauses when tab is hidden
- [ ] No `MARKET_DATA` mock import remains in `app/market/page.tsx`
- [ ] Lighthouse: `/market` initial paint < 2.5s, no console errors
- [ ] Playwright suite green

---

## 12. What to Do Next (Action Sequence)

1. ✅ Confirm API plan ([LIVE_MARKET_DATA_API_IMPLEMENTATION.md](LIVE_MARKET_DATA_API_IMPLEMENTATION.md)) is approved and Phase P1–P6 are merged.
2. ⏳ Phase **W1** — add `market` namespace to `@quantts/api-client`, generate types.
3. ⏳ Phase **W2** — add hooks; unit-test against a mock fetch.
4. ⏳ Phase **W3** — swap `MARKET_DATA` → real watchlist on `/market`.
5. ⏳ Phase **W4 + W5** — build chart + drill-down (parallelizable).
6. ⏳ Phase **W6 + W7 + W8** — Dashboard + Agent Detail + News/Sentiment (parallelizable).
7. ⏳ Phase **W9 + W10** — polish + tests, then ship to staging.
8. 🔜 Phase **W11** — SSE live ticks once Phase 2 of API plan is delivered.

---

## 13. Notes on Existing Code

- **Don't delete `lib/mock-data.ts`** — other tests still reference it. Just stop importing `MARKET_DATA` in page files.
- **Reuse existing `getClient()`** in `lib/client.ts` — every route handler should go through it (consistent cookie + base URL).
- **Match existing styling** — `var(--color-bg-panel)`, `var(--color-border-subtle)`, etc. (no Tailwind / no new CSS framework).
- **Keep the existing `/live` page intact for now** — telemetry feed lives there; market data is a separate surface.

---

## 14. Done When

- [ ] Real market data visible on `/dashboard`, `/market`, `/market/[symbol]`, `/agents/[id]`
- [ ] Mock `MARKET_DATA` import removed from production pages
- [ ] All 6 hooks + 6 components + 6 route handlers shipped
- [ ] Tests green (unit + Playwright)
- [ ] Staging smoke test passed
- [ ] Web team handoff doc updated in `apps/web/README.md`
