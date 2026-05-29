# AI Agent + Telemetry — Implementation Plan

> **Scope:** End-to-end implementation of the 4-analyst AI agent pipeline with real-time telemetry streaming to web and mobile clients.
> **Goal:** Take the agent flow from `mock_llm` + `demo_market` to a production-ready, LLM-driven, real-data-backed agent with full live telemetry visibility.
> **References:** `ProjectScope.txt` §6.2 (AI Trading Engine), §6.7 (Real-Time Telemetry); `ClientApprovedDoc.txt` mobile §5 (Live Tab).

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│  PYTHON AI ENGINE  (services/quantt-backend)                 │
│                                                              │
│  Orchestration Graph (graph.py)                              │
│   ├─ Fundamental Analyst  ──┐                                │
│   ├─ Sentiment Analyst    ──┤ (parallel)                     │
│   ├─ News Analyst         ──┤                                │
│   └─ Technical Analyst    ──┘                                │
│         │                                                    │
│         ▼                                                    │
│   Research Lead  (debate + consensus)                        │
│         │                                                    │
│         ▼                                                    │
│   Trader Agent  (decision: buy/sell/hold)                    │
│         │                                                    │
│         ▼                                                    │
│   Risk Manager  (validate vs user rules)                     │
│         │                                                    │
│         ▼                                                    │
│   Execution Router  (best-price across DEXes)                │
│         │                                                    │
│         ▼                                                    │
│   On-Chain Commit  (Lithosphere audit trail)                 │
│                                                              │
│  ALL STEPS → TelemetryPublisher.publish() → Redis            │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ Redis pub/sub channel
                       `quantt.telemetry.v1`
                              │
┌──────────────────────────────────────────────────────────────┐
│  NODE API GATEWAY  (apps/api)                                │
│   - telemetry-bus.ts plugin (subscribes to Redis)            │
│   - GET /v1/telemetry/stream  (SSE endpoint)                 │
└──────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌──────────────────────┐         ┌──────────────────────┐
│  WEB CLIENT          │         │  MOBILE CLIENT       │
│  (apps/web)          │         │  (apps/mobile)       │
│  EventSource (SSE)   │         │  react-native-sse    │
│  → Dashboard 4.2     │         │  → Live Tab 5.4      │
│  → Agent Detail 4.5  │         │  → Home Dash 5.2     │
└──────────────────────┘         └──────────────────────┘
```

---

## 2. Current Codebase State

### ✅ Already in place

| Component | Location | Status |
|-----------|----------|--------|
| 4 Analysts (skeleton) | `services/quantt-backend/quantt/agents/analysts.py` | Files exist, need real LLM wiring |
| Research Lead | `services/quantt-backend/quantt/agents/research.py` | Skeleton exists |
| Trader Agent | `services/quantt-backend/quantt/agents/trader.py` | Skeleton exists |
| Risk Manager | `services/quantt-backend/quantt/agents/risk.py` | Skeleton exists |
| Orchestration Graph | `services/quantt-backend/quantt/orchestration/graph.py` | Wires analysts → research → trader → risk |
| Telemetry Publisher (Python) | `services/quantt-backend/quantt/telemetry/publisher.py` | Publishes JSON events to Redis |
| Telemetry Bus (Node) | `apps/api/src/plugins/telemetry-bus.ts` | Subscribes to Redis channel |
| SSE Endpoint | `apps/api/src/routes/dashboard.ts` (`/v1/telemetry/stream`) | Streams events to clients |
| Mobile Live Tab | `apps/mobile/app/(tabs)/live.tsx` | Renders telemetry events from `useApp()` |
| Mock LLM | `services/quantt-backend/quantt/providers/mock_llm.py` | **To be replaced with OpenAI** |
| Demo market data | `services/quantt-backend/quantt/providers/demo_market.py` | **To be replaced with real APIs** |

### ❌ Missing / To Build

| Component | Why It's Needed |
|-----------|-----------------|
| Real LLM provider (OpenAI GPT-4) | Analysts and research lead must call real model |
| Real market data provider | Live prices, OHLCV, volume |
| News + sentiment data feed | For News Analyst + Sentiment Analyst |
| Indicator computation (MACD/RSI) | For Technical Analyst |
| Execution Router (DEX best-price) | Quote all DEXes, pick cheapest |
| Wallet signing module | Sign and submit trade transactions |
| Lithosphere audit-commit module | Hash decisions and submit on-chain |
| Telemetry events at every step | Currently many flow steps are silent |
| Web SSE consumer hook | Dashboard + Agent Detail must subscribe |
| Mobile event filtering by agent | Live Tab today shows all; agent screen needs per-agent filter |

---

## 3. Implementation Phases

### Phase 1 — Wire Real LLM (1 day)

**File:** `services/quantt-backend/quantt/providers/`

1. Create `openai_llm.py` implementing same interface as `mock_llm.py`.
2. Read `OPENAI_API_KEY` from `settings.py`.
3. Add provider switch in `settings.py`: `LLM_PROVIDER=openai|mock`.
4. Update each analyst to use `prompts.py` templates with the real LLM.

**Telemetry events emitted:**
- `analyst.started` — when each analyst begins
- `analyst.completed` — with verdict (bullish/bearish/neutral + confidence)

---

### Phase 2 — Wire Real Market Data (2 days)

**File:** `services/quantt-backend/quantt/providers/market_data.py` (new)

> **Blocker:** Awaiting client confirmation on provider choice (Binance / GeckoTerminal / Lithosphere RPC).

**Sub-modules:**
- `cex_provider.py` — Binance WebSocket for BTC/ETH/major coins
- `dex_provider.py` — GeckoTerminal API for Arbitrum / Base / BNB pool prices
- `lithosphere_provider.py` — Direct RPC + Lithosphere DEX contract reads
- `indicators.py` — MACD, RSI computed with `pandas-ta`
- `news_provider.py` — CryptoPanic API integration
- `sentiment_provider.py` — GPT-4-based scoring on news + social

Each analyst pulls only what it needs:
- Fundamental → fundamentals API (TBD) or hardcoded project data
- Technical → OHLCV + indicators
- Sentiment → sentiment score
- News → news feed

---

### Phase 3 — Telemetry Coverage (1 day)

Add `TelemetryPublisher.publish(...)` calls at every meaningful step of `orchestration/graph.py` and downstream agents.

**Event schema (already defined in `publisher.py`):**

```json
{
  "id": "evt-2026-05-21T10:30:01Z",
  "type": "analyst.completed",
  "level": "info",
  "title": "Fundamental Analyst",
  "detail": "ETH fundamentals strong — TVL up 12% MoM",
  "source": "quantt-backend",
  "timestamp": "2026-05-21T10:30:01Z",
  "metadata": {
    "agentId": "ag_123",
    "symbol": "ETH",
    "analyst": "fundamental",
    "verdict": "bullish",
    "confidence": 0.82
  }
}
```

**Events to emit (full list):**

| Stage | Event Type | When |
|-------|-----------|------|
| Agent start | `agent.started` | Run begins |
| Market data fetch | `market.snapshot` | After data pulled |
| Analyst start | `analyst.started` | Each of 4 analysts begins |
| Analyst result | `analyst.completed` | Each returns its verdict |
| Debate | `research.debate` | Research Lead reasoning step |
| Consensus | `research.consensus` | Final balanced view |
| Trade intent | `trader.decision` | Buy/sell/hold + size + reason |
| Risk check | `risk.check.started` | Risk validation begins |
| Risk verdict | `risk.approved` / `risk.rejected` | With per-rule breakdown |
| DEX quotes | `execution.quotes` | All DEX quotes returned |
| DEX choice | `execution.routed` | Best DEX selected |
| Tx submitted | `execution.submitted` | tx_hash assigned |
| Tx confirmed | `execution.confirmed` | On-chain confirmation |
| Audit commit | `audit.committed` | Lithosphere hash stored |
| Agent end | `agent.completed` | Run ends |

Always include `agentId` in `metadata` so clients can filter per agent.

---

### Phase 4 — Execution Router (2 days)

**File:** `services/quantt-backend/quantt/execution/` (new)

- `router.py` — Parallel quote requests across supported DEXes
- `dex_adapters/uniswap_v3.py` — Quoter contract calls (Base, Arbitrum)
- `dex_adapters/sushiswap.py` — Quoter calls (Base, Arbitrum, BNB)
- `dex_adapters/pancakeswap.py` — BNB Chain
- `dex_adapters/lithosphere_dex.py` — Lithosphere native DEX
- `wallet.py` — Sign and submit txs via `viem` equivalent (`web3.py` + private key from KMS / env)
- `confirmation.py` — Poll for receipt, store `tx_hash`

---

### Phase 5 — On-Chain Audit Commit (1 day)

**File:** `services/quantt-backend/quantt/audit/lithosphere_commit.py`

- Take all telemetry events from a run
- Hash with SHA-256
- Submit hash to a Lithosphere smart contract (`AuditLog.commit(bytes32 hash)`)
- Save `tx_hash` back in DB against the agent run record

Emits `audit.committed` telemetry event with the on-chain `tx_hash`.

---

### Phase 6 — Web Client Integration (1 day)

**Files:**
- `apps/web/src/hooks/useTelemetry.ts` — wraps `EventSource('/v1/telemetry/stream')`
- `apps/web/src/components/ActivityFeed.tsx` — Dashboard 4.2 stream display
- `apps/web/src/app/agents/[id]/page.tsx` — Per-agent telemetry filter (metadata.agentId === id)

---

### Phase 7 — Mobile Client Polish (0.5 day)

**Files:**
- `apps/mobile/app/(tabs)/live.tsx` — already exists, verify event rendering matches new event types
- `apps/mobile/app/agent/[id].tsx` — add per-agent telemetry filter
- Install `react-native-sse` if not already (the existing `useApp()` may already handle this — verify)

---

## 4. Dependencies & Blockers

| Blocker | Owner | Status |
|---------|-------|--------|
| Client confirmation: market data provider | Client | ⏳ Awaiting reply |
| Client confirmation: Lithosphere price source | Client | ⏳ Awaiting reply |
| OpenAI API key provisioning | DevOps | ❌ Needed |
| Lithosphere RPC endpoint URL | DevOps / Lithosphere team | ❌ Needed |
| AuditLog smart contract deployed on Lithosphere | Blockchain team | ❌ Needed |
| Agent wallet key management (KMS / env) | DevOps | ❌ Needed |

---

## 5. Environment Variables to Add

```bash
# AI
OPENAI_API_KEY=
LLM_PROVIDER=openai

# Market data
BINANCE_WS_URL=wss://stream.binance.com:9443
GECKOTERMINAL_API_URL=https://api.geckoterminal.com/api/v2
CRYPTOPANIC_API_KEY=

# Blockchain
ARBITRUM_RPC_URL=
BASE_RPC_URL=
BNB_RPC_URL=
LITHOSPHERE_RPC_URL=
AUDIT_LOG_CONTRACT_ADDRESS=

# Wallet (use KMS in prod)
AGENT_WALLET_PRIVATE_KEY=

# Telemetry (already present)
REDIS_URL=
TELEMETRY_CHANNEL=quantt.telemetry.v1
```

---

## 6. Effort Estimate

| Phase | Days | Risk |
|-------|------|------|
| 1. Real LLM | 1 | Low |
| 2. Real market data | 2 | **High** (blocked on client) |
| 3. Telemetry coverage | 1 | Low |
| 4. Execution router | 2 | Medium |
| 5. On-chain commit | 1 | Medium (depends on contract) |
| 6. Web integration | 1 | Low |
| 7. Mobile polish | 0.5 | Low |
| **Total** | **8.5 days** | |

---

## 7. Acceptance Criteria

- [ ] Run `python -m quantt run --symbol ETH --agent ag_test` and see all 15 telemetry events fire in order
- [ ] Web Dashboard activity feed updates within 1 sec of each event
- [ ] Mobile Live Tab updates within 1 sec of each event
- [ ] Mobile Agent Detail screen filters telemetry to that agent only
- [ ] At least one full trade executes end-to-end against a testnet DEX
- [ ] `audit.committed` event includes a real Lithosphere `tx_hash`
- [ ] Risk rejection path tested — a violating trade produces a `risk.rejected` event and the trade does NOT execute
- [ ] Reconnection works — closing and re-opening the SSE stream resumes event flow

---

## 8. What to Do Next (Immediate Actions)

1. **Get client answers** on the 4 market-data questions (sent on WhatsApp).
2. **Provision OpenAI API key** + add to env.
3. **Start Phase 1 (Real LLM)** — unblocked, can begin immediately.
4. **Start Phase 3 (Telemetry coverage)** — unblocked, can run in parallel with Phase 1.
5. **Pause Phases 2, 4, 5** until client + DevOps unblock dependencies.
