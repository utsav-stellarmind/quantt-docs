# QUANTT Platform — Implementation Status Document

> **Last Updated:** 2026-05-19
> **Branch:** signup
> **Prepared by:** AI Audit of `d:\stellarmind\Quantt-Dev`

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Tech Stack](#2-tech-stack)
3. [What Is Already Implemented](#3-what-is-already-implemented)
   - [Authentication & Session](#31-authentication--session)
   - [Web App (Next.js)](#32-web-app-nextjs)
   - [Mobile App (Expo / React Native)](#33-mobile-app-expo--react-native)
   - [API Gateway (Fastify)](#34-api-gateway-fastify)
   - [Python AI Backend](#35-python-ai-backend)
   - [Shared Packages](#36-shared-packages)
   - [Infrastructure & DevOps](#37-infrastructure--devops)
4. [What Is Missing / Not Yet Implemented](#4-what-is-missing--not-yet-implemented)
   - [Authentication Gaps](#41-authentication-gaps)
   - [Web App Gaps](#42-web-app-gaps)
   - [Mobile App Gaps](#43-mobile-app-gaps)
   - [API Gateway Gaps](#44-api-gateway-gaps)
   - [AI Trading Engine Gaps](#45-ai-trading-engine-gaps)
   - [Blockchain & DEX Integration Gaps](#46-blockchain--dex-integration-gaps)
   - [Enterprise & B2B Gaps](#47-enterprise--b2b-gaps)
   - [Billing & Payments Gaps](#48-billing--payments-gaps)
   - [Infrastructure Gaps](#49-infrastructure-gaps)
5. [Feature-by-Feature Status Matrix](#5-feature-by-feature-status-matrix)
6. [Database Schema Status](#6-database-schema-status)
7. [API Endpoint Inventory](#7-api-endpoint-inventory)
8. [Priority Build Order](#8-priority-build-order)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│   apps/web (Next.js 15)        apps/mobile (Expo 52 / RN 0.76) │
└────────────────────────┬───────────────────────┬────────────────┘
                         │ HTTP + SSE             │ HTTP + SSE
                         ▼                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                 apps/api — Fastify 5 BFF                        │
│  Auth │ Dashboard │ Mobile Endpoints │ Telemetry SSE            │
└───────────────┬─────────────────────────────┬───────────────────┘
                │ HTTP proxy                  │ Redis pub/sub
                ▼                             ▼
┌───────────────────────────┐   ┌─────────────────────────────────┐
│  services/quantt-backend  │   │           Redis 7               │
│  Python FastAPI            │   │  Nonces │ Tokens │ Telemetry    │
│  AI Engine & Market Data   │   └─────────────────────────────────┘
└───────────────┬───────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      PostgreSQL 16                              │
│  Users │ RefreshTokens │ TelemetryCursor                        │
└─────────────────────────────────────────────────────────────────┘
```

**Monorepo layout:**

```
Quantt-Dev/
├── apps/
│   ├── web/          — Next.js operator portal
│   ├── mobile/       — Expo React Native app
│   └── api/          — Fastify BFF + auth gateway
├── packages/
│   ├── types/        — Shared TypeScript interfaces
│   ├── api-client/   — HTTP client library
│   ├── ui-web/       — Web component library
│   ├── ui-mobile/    — React Native component library
│   └── design-tokens/— Unified color / spacing / typography
├── services/
│   └── quantt-backend/ — Python FastAPI AI engine
└── deploy/
    └── secrets/      — Docker secret files
```

---

## 2. Tech Stack

| Layer | Technology | Version | Status |
|-------|-----------|---------|--------|
| Web Frontend | Next.js + React + TypeScript | 15 / 18 / 5.7 | Implemented |
| Mobile Frontend | Expo + React Native + TypeScript | 52 / 0.76 / 5.7 | Implemented |
| API Gateway | Fastify + TypeScript | 5.2.1 | Implemented |
| AI Engine | FastAPI + Python | 3.11 | Partially implemented |
| Database | PostgreSQL + Prisma ORM | 16 / 6.5 | Partial schema |
| Cache / Realtime | Redis 7 | 7 | Implemented |
| LLM Provider | OpenAI GPT-4 | — | Not wired in Python backend |
| Blockchain RPC | Lithosphere EVM (Chain ID 1890) | — | Config only |
| Auth | JWT + bcrypt + viem SIWE | — | Implemented |
| Payments | QUANTT ERC-20 token | — | UI stubbed, no on-chain logic |
| Containerisation | Docker + Docker Compose | — | Implemented |

---

## 3. What Is Already Implemented

### 3.1 Authentication & Session

**Fully working end-to-end:**

- **Email / Password Registration** (`POST /v1/auth/register`)
  - Accepts `name`, `email`, `password` (min 10 chars)
  - bcrypt hashing, uniqueness check, role default = `RETAIL_TRADER`
  - Issues JWT access token (15 min) + refresh token (30 days)
  - Tokens stored in `httpOnly` cookies

- **Email / Password Login** (`POST /v1/auth/login`)
  - bcrypt compare, brute-force rate limit (10 req/min)
  - Issues new JWT pair

- **Token Refresh** (`POST /v1/auth/refresh`)
  - Single-use refresh tokens with rotation
  - Hashed in `RefreshToken` table, revocation tracked

- **EVM Wallet Auth** (2-step: challenge → verify)
  - `POST /v1/auth/wallet/challenge` — generates SIWE-style message, nonce stored in Redis (300 s TTL)
  - `POST /v1/auth/wallet/verify` — verifies signature via viem `verifyMessage`, issues JWT pair
  - Chain ID 1890 (Lithosphere) enforced

- **Mobile biometric unlock** — Face ID / Touch ID using expo-local-authentication; session stored in AsyncStorage with encryption flag

**Not yet implemented in Auth:**
- Email verification (no OTP / magic link flow)
- Password reset flow
- 2FA / TOTP
- SSO (OAuth 2.0 / Google / Apple)
- Role-based middleware guard on protected routes (guard is missing — any authenticated user can call any endpoint)

---

### 3.2 Web App (Next.js)

**Implemented screens / pages:**

| Route | What It Does | Status |
|-------|-------------|--------|
| `/` | Redirect to `/dashboard` or `/login` | Done |
| `/(marketing)` | Landing / marketing page | Done |
| `/login` | Email+password form + wallet connect button | Done |
| `/dashboard` | Portfolio equity, Sharpe, drawdown, active agents, agent table, activity feed | Done |
| `/agents/[id]` | Agent detail — strategy, chain, status, PnL, exposure, configuration summary | Done |

**Implemented components:**
- `MetricCard` — KPI with label, value, delta
- `AgentTable` — sortable list of active agents
- `ActivityFeed` — live telemetry event stream via SSE
- `Nav` — header navigation

**Real-time telemetry:** SSE from `GET /v1/telemetry/stream` (Redis pub/sub) → live feed on dashboard.

**Missing web screens (see Section 4.2):** Create Agent, Edit Agent, Portfolio deep-dive, Billing, Admin Dashboard, Enterprise Admin, Marketplace, Settings.

---

### 3.3 Mobile App (Expo / React Native)

**Implemented screens:**

| Screen | File | Status |
|--------|------|--------|
| Auth gate / splash | `app/index.tsx` | Done |
| Login (email + wallet modes) | `app/login.tsx` | Done |
| Home dashboard | `app/(tabs)/index.tsx` | Done |
| Agents list | `app/(tabs)/agents.tsx` | Done |
| Create agent | `app/agent/create.tsx` | Done |
| Agent detail | `app/agent/[id].tsx` | Done |
| Advanced control surface | `app/(tabs)/control.tsx` | Done |
| Insights (win rate, model confidence, reasoning) | `app/(tabs)/insights.tsx` | Done |
| Portfolio (positions, balances, chain breakdown) | `app/(tabs)/portfolio.tsx` | Done |
| Live feed (real-time trade stream) | `app/(tabs)/live.tsx` | Done |
| Alerts | `app/alerts.tsx` | Done |
| Wallet (QUANTT balance + transactions) | `app/wallet.tsx` | Done |
| Billing (invoices + usage costs) | `app/billing.tsx` | Done |
| Copilot (AI chat) | `app/copilot.tsx` | Done (rule-based demo) |
| Marketplace (strategy templates) | `app/marketplace.tsx` | Done (static data) |
| Enterprise (fleet health, nodes) | `app/enterprise.tsx` | Done (demo data) |

**Implemented services / infrastructure:**
- `src/services/biometrics.ts` — Face ID / Touch ID unlock
- `src/services/notifications.ts` — Local push alerts; Expo Notifications integration
- `src/services/storage.ts` — AsyncStorage session persistence and cache
- `src/app-provider.tsx` — Global state context + hooks
- `src/client.ts` — API client with auth headers
- Telemetry streaming via SSE with 45 s polling fallback

**Missing mobile items (see Section 4.3):** Real push notifications via APNs/FCM backend, actual 2FA, Settings screen with granular preferences.

---

### 3.4 API Gateway (Fastify)

**All implemented endpoints:**

```
POST   /v1/auth/register
POST   /v1/auth/login
POST   /v1/auth/refresh
POST   /v1/auth/wallet/challenge
POST   /v1/auth/wallet/verify

GET    /v1/dashboard
GET    /v1/agents/:id
GET    /v1/telemetry
GET    /v1/telemetry/stream          ← SSE

GET    /v1/mobile/overview
GET    /v1/mobile/agents/:id
POST   /v1/mobile/agents
POST   /v1/mobile/agents/:id/state
GET    /v1/mobile/alerts
POST   /v1/mobile/alerts/:id/ack
GET    /v1/mobile/wallet
GET    /v1/mobile/billing
GET    /v1/mobile/marketplace
GET    /v1/mobile/social
GET    /v1/mobile/enterprise
POST   /v1/mobile/wallet/pay
POST   /v1/mobile/copilot
```

**Infrastructure plugins:**
- `plugins/prisma.ts` — Prisma client as Fastify decorator
- `plugins/redis.ts` — Redis client as Fastify decorator
- `plugins/security.ts` — Helmet headers + rate limiting
- `plugins/telemetry-bus.ts` — Redis pub/sub subscriber for SSE forwarding

**Demo data fallback:** `lib/demo-data.ts` and `lib/mobile-data.ts` return realistic fake data when Python backend is unreachable — allows frontend development without AI engine running.

---

### 3.5 Python AI Backend

**What exists:**
- FastAPI app running on port 8000
- Health check endpoint (`/ready`)
- Proxy endpoints: `/dashboard`, `/agents/:id`, `/telemetry`
- Connected to Lithosphere RPC, PostgreSQL, Redis via environment config

**What is NOT wired up yet** (see Section 4.5 for full detail):
- Actual AI analyst agents (Fundamental, Sentiment, News, Technical)
- Research Lead consensus layer
- Trader Agent decision engine
- Risk Manager validation logic
- DEX execution and price routing
- Lithosphere blockchain audit trail writes
- OpenAI GPT-4 calls
- Historical backtesting engine

---

### 3.6 Shared Packages

**`packages/types`** — Complete TypeScript interfaces for all data models:
- `AgentStatus`, `RiskLevel`, `ActivityType`
- `PortfolioSummary`, `AgentSummary`, `AgentDetail`
- `AlertItem`, `RiskSnapshot`, `TradeExecution`
- `WalletSnapshot`, `BillingSnapshot`
- `DashboardPayload`, `MobileOverviewPayload`
- Strategy, Position, Marketplace, Social, Enterprise types (20+ total)

**`packages/api-client`** — `QuanttsApiClient` class:
- Bearer token injection, logging, error handling
- Methods for all auth, dashboard, and mobile endpoints

**`packages/ui-web`** — Minimal: `Button`, `Card` (other components inline in pages)

**`packages/ui-mobile`** — Full component set:
- `ActionButton`, `MetricCard`, `InputField`, `OptionChips`, `StatusPill`
- `RiskMeter`, `DataRow`, `MiniBars`, `SectionHeader`, `Panel`

**`packages/design-tokens`** — Dark-mode design system:
- Colors, spacing, radii, typography — fully defined

---

### 3.7 Infrastructure & DevOps

- **Docker Compose** — Orchestrates postgres, redis, quantt-backend, api, web
- **Secret management** — `deploy/secrets/` with `*_FILE` env var pattern for Docker secrets
- **Environment validation** — Zod schema in `apps/api/src/config/env.ts`
- **Database migrations** — Prisma migration system in place

---

## 4. What Is Missing / Not Yet Implemented

### 4.1 Authentication Gaps

| Feature | Scope Doc Requirement | Current State | Priority |
|---------|----------------------|---------------|----------|
| Email verification | Required for registration | Not implemented — account created without verifying email | High |
| Password reset / forgot password | Required | Not implemented | High |
| 2FA / TOTP | Required (SSO and 2FA listed in scope) | Not implemented | High |
| SSO (Google, Apple, etc.) | In scope | Not started | Medium |
| Role-based route guards | RBAC — users should only access what their role allows | No middleware enforcing role on API routes | High |
| Logout / token revocation endpoint | Basic hygiene | No `POST /v1/auth/logout` endpoint | Medium |
| Session list & revoke all | Security feature | Not implemented | Low |

---

### 4.2 Web App Gaps

| Screen / Feature | Scope Requirement | Current State | Priority |
|-----------------|------------------|---------------|----------|
| Create Agent page | 4.4 — full agent form with all config fields | Not implemented — web only has view, no create UI | High |
| Edit Agent page | 4.5 — edit agent config | Not implemented | High |
| Agent pause / stop / kill controls | 4.5 — Start/Pause/Stop/Edit controls | Not implemented in web | High |
| Emergency kill switch (all agents) | 4.5 — Emergency kill switch | Not implemented in web | High |
| Portfolio deep-dive page | 4.6 — equity, open positions, chains, drawdown, Sharpe, win rate | Not implemented — only summary on dashboard | High |
| Billing & Subscription page | 4.7 — plan, invoices, usage, QUANTT token billing | Not implemented in web | High |
| Enterprise Admin Dashboard | 4.8 — tenant management, API keys, user roles, invoices | Not implemented | High |
| Settings page | Profile, wallet, notifications, 2FA | Not implemented | Medium |
| Marketplace page | Browse strategy templates | Not implemented in web | Medium |
| Live telemetry page (dedicated) | 4.5 agent detail live tab | Partially in activity feed only | Medium |
| Performance charts (visual) | 4.6 — performance charts, drawdown history | No charting library integrated | Medium |
| OHLCV candlestick charts | 4.3 — market data with charts, MACD, RSI | Not implemented | Medium |
| Copilot / AI assistant | 5.10 — chat interface | Not implemented in web | Low |
| Social / leaderboard page | 5.11 marketplace leaderboard | Not implemented | Low |
| Admin: User management CRUD | 4.8 — create/delete users, assign roles | Not implemented | High |
| Admin: API key management | 4.8 — generate, revoke API keys | Not implemented | High |

---

### 4.3 Mobile App Gaps

| Feature | Scope Requirement | Current State | Priority |
|---------|------------------|---------------|----------|
| Push notifications (server-side) | 5.9 — APNs/FCM delivery from backend | Client code exists; backend does not send pushes | High |
| Settings screen | 5.13 — connected wallets, notification prefs, risk rules, kill switch | Not implemented as dedicated screen | Medium |
| 2FA setup screen | 5.1 — 2FA | No 2FA UI or setup flow | High |
| Real Copilot responses | 5.10 — AI responses about portfolio, reasoning | Currently rule-based demo strings; not connected to LLM | High |
| Real Marketplace (purchase / subscribe) | 5.11 — subscribe and deploy from template | Currently static listing, no transaction flow | Medium |
| Strategy backtesting trigger | Scope 6.2 — Pro/Admin/Enterprise backtesting | Not implemented | Medium |
| Coordination graph (visual) | Insights screen mentions coordination graph | No graph visualisation implemented | Low |
| Social leaderboard detail | 5.11 — social leaderboard | Basic list only | Low |

---

### 4.4 API Gateway Gaps

| Endpoint / Feature | Required By | Current State | Priority |
|-------------------|------------|---------------|----------|
| `POST /v1/auth/logout` | Revoke refresh token, clear cookies | Not implemented | High |
| `POST /v1/auth/forgot-password` | Password reset flow | Not implemented | High |
| `POST /v1/auth/reset-password` | Token-based password reset | Not implemented | High |
| `POST /v1/auth/verify-email` | Email OTP verification | Not implemented | High |
| `POST /v1/auth/2fa/setup` + `/verify` | TOTP 2FA | Not implemented | High |
| Role guard middleware | RBAC enforcement on all protected routes | Not implemented — any valid JWT can call any route | High |
| `GET /v1/admin/users` | Admin user list | Not implemented | High |
| `POST /v1/admin/users` | Admin create user | Not implemented | High |
| `DELETE /v1/admin/users/:id` | Admin delete user | Not implemented | High |
| `PATCH /v1/admin/users/:id/role` | Admin assign role | Not implemented | High |
| `GET /v1/admin/organizations` | Enterprise tenant list | Not implemented | High |
| `POST /v1/admin/api-keys` | Generate API key for enterprise | Not implemented | High |
| `DELETE /v1/admin/api-keys/:id` | Revoke API key | Not implemented | High |
| `GET /v1/admin/invoices` | Admin billing overview | Not implemented | High |
| `GET /v1/portfolio` | Full portfolio with open positions | Not implemented (dashboard returns summary only) | High |
| `GET /v1/market/:symbol` | Live price, OHLCV, indicators | Not implemented | Medium |
| `GET /v1/agents/:id/backtest` | Backtesting trigger/results | Not implemented | Medium |
| `POST /v1/notifications/push` | Send push to device via APNs/FCM | Not implemented | High |
| `GET /v1/billing` | Web billing page data | Not implemented | High |
| `POST /v1/billing/invoice/:id/pay` | On-chain invoice payment | Not implemented | High |
| `GET /v1/enterprise` | Enterprise fleet data for web | Not implemented (mobile-only) | Medium |
| `GET /v1/marketplace` | Strategy templates for web | Not implemented (mobile-only endpoint exists) | Medium |
| Rate limiting on all routes | Currently only auth routes rate-limited | Non-auth routes have no rate limiting | Medium |
| Request logging / audit trail | Compliance requirement | Not implemented | Medium |

---

### 4.5 AI Trading Engine Gaps

This is the largest gap. The Python `quantt-backend` service exists but the actual AI logic is not implemented.

| Component | Description | Status | Priority |
|-----------|-------------|--------|----------|
| **Fundamental Analyst Agent** | LLM agent that analyses token fundamentals (revenue, adoption, growth) | Not implemented | Critical |
| **Sentiment Analyst Agent** | LLM agent that reads market sentiment (fear/greed index, social signals) | Not implemented | Critical |
| **News Analyst Agent** | LLM agent that reads and interprets news feed for price-impacting events | Not implemented | Critical |
| **Technical Analyst Agent** | Computes and interprets MACD, RSI, price patterns, volume | Not implemented | Critical |
| **Parallel execution harness** | Run all 4 analysts simultaneously and collect results | Not implemented | Critical |
| **Research Lead Agent** | LLM agent that debates and synthesises 4 analyst opinions | Not implemented | Critical |
| **Trader Agent** | LLM agent that makes Buy/Sell/Hold decision with size and rationale | Not implemented | Critical |
| **Risk Manager** | Validates trade decision against user's risk rules (max loss, max position, drawdown) | Not implemented | Critical |
| **OpenAI GPT-4 integration** | LLM API calls for all above agents | Not integrated | Critical |
| **Live market data feed** | Real-time price and volume from exchange APIs (CoinGecko, Binance, etc.) | Not implemented | Critical |
| **OHLCV data provider** | Historical candlestick data for technical analysis | Not implemented | Critical |
| **News feed integration** | Live news API (CryptoPanic, NewsAPI, etc.) | Not implemented | Critical |
| **Social sentiment provider** | Sentiment score API (LunarCrush, Santiment, etc.) | Not implemented | Critical |
| **MACD / RSI calculation** | Technical indicator engine | Not implemented | High |
| **Agent scheduler** | Scheduled + event-driven agent triggers | Not implemented | High |
| **Event-driven triggers** | Trigger agent on price move > X% | Not implemented | High |
| **Backtesting engine** | Historical simulation of strategy | Not implemented | Medium |
| **Audit trail writer** | Hash and write every decision to Lithosphere blockchain | Not implemented | High |
| **Telemetry publisher** | Publish AI decision events to Redis `quantt.telemetry.v1` channel | Not implemented | High |

---

### 4.6 Blockchain & DEX Integration Gaps

| Feature | Description | Status | Priority |
|---------|-------------|--------|----------|
| **Agent wallet creation** | Each agent needs its own EVM wallet for signing trades | Not implemented | Critical |
| **Uniswap V3 integration** | Price quotes + swap execution on Base and Arbitrum | Not implemented | Critical |
| **SushiSwap integration** | Price quotes + swap execution on Base, Arbitrum, BNB Chain | Not implemented | Critical |
| **PancakeSwap integration** | Price quotes + swap execution on BNB Chain | Not implemented | Critical |
| **Lithosphere DEX integration** | Price quotes + swap execution on LITHO chain | Not implemented (marked in-progress in scope) | High |
| **Best price router** | Query all available DEXes simultaneously, pick best price + lowest fee | Not implemented | Critical |
| **Transaction signing** | Sign trade transaction with agent wallet | Not implemented | Critical |
| **On-chain submission** | Submit signed transaction to appropriate chain RPC | Not implemented | Critical |
| **Transaction confirmation tracking** | Wait for tx confirmation, store `tx_hash` | Not implemented | Critical |
| **Lithosphere audit trail** | Hash every AI decision and write to Lithosphere blockchain | Not implemented | High |
| **ERC-20 QUANTT token payment** | Verify on-chain payment for invoices | Not implemented | High |
| **Chain selector** | Auto-select chain based on agent config and token availability | Not implemented | High |
| **Slippage control** | Set slippage tolerance per trade | Not implemented | High |
| **Gas estimation** | Estimate gas before submitting | Not implemented | High |

---

### 4.7 Enterprise & B2B Gaps

| Feature | Description | Status | Priority |
|---------|-------------|--------|----------|
| Organisation model in DB | No `Organisation`, `OrgMember`, `ApiKey` tables in Prisma schema | Not implemented | High |
| Organisation registration | B2B org sign-up flow | Not implemented | High |
| API key generation | SHA-256 hashed keys with scopes | Not implemented | High |
| API key authentication | Verify API key on inbound requests | Not implemented | High |
| Sub-user management | Owner / Member / Viewer within org | Not implemented | High |
| Usage metering | Count API calls per org per key | Not implemented | High |
| Redis token-bucket rate limiting per key | Per-API-key rate limits | Not implemented | High |
| Monthly invoice generation | Auto-generate invoices from usage meters | Not implemented | High |
| On-chain payment verification | Verify ERC-20 QUANTT payment for invoices | Not implemented | High |
| Webhook reconciliation | Notify org on payment confirmed | Not implemented | Medium |
| Enterprise admin web dashboard | Tenant management UI | Not implemented | High |

---

### 4.8 Billing & Payments Gaps

| Feature | Description | Status | Priority |
|---------|-------------|--------|----------|
| Subscription plan table in DB | No `Plan`, `Subscription` tables | Not implemented | High |
| Subscription plan selection UI (web) | 4.7 — billing page | Not implemented | High |
| Usage cost calculation | Compute cost from API calls + agent runtime | Not implemented | High |
| Invoice generation (auto) | Create invoice records at billing cycle | Not implemented | High |
| QUANTT token balance read from chain | Read ERC-20 balance from wallet on-chain | Not implemented | High |
| On-chain payment flow | Pay invoice by submitting ERC-20 transfer on-chain | Not implemented | High |
| Invoice status webhook | Mark invoice paid after on-chain confirmation | Not implemented | High |
| Plan upgrade self-serve | User upgrades plan without admin | Not implemented | Medium |

---

### 4.9 Infrastructure Gaps

| Feature | Description | Status | Priority |
|---------|-------------|--------|----------|
| Push notification service | APNs / FCM integration to send alerts from backend | Not implemented | High |
| Email service | Send verification, reset, invoice emails | Not implemented | High |
| Monitoring / observability | No Prometheus metrics, no Grafana, no Sentry | Not implemented | Medium |
| CI/CD pipeline | No GitHub Actions or equivalent pipelines | Not implemented | Medium |
| Secrets rotation | No mechanism to rotate JWT secrets in prod | Not implemented | Medium |
| Production TLS termination | Nginx / Caddy reverse proxy config | Not implemented | Medium |
| Horizontal scaling config | Redis cluster, Prisma connection pooling (PgBouncer) | Not implemented | Low |

---

## 5. Feature-by-Feature Status Matrix

### Authentication

| Feature | Web | Mobile | API Backend | Complete? |
|---------|-----|--------|-------------|-----------|
| Email/password registration | ✓ | ✓ | ✓ | Yes |
| Email/password login | ✓ | ✓ | ✓ | Yes |
| JWT access + refresh tokens | ✓ | ✓ | ✓ | Yes |
| EVM wallet challenge/verify | ✓ | ✓ | ✓ | Yes |
| Biometric unlock (Face ID / Touch ID) | — | ✓ | — | Mobile only |
| Email verification | ✗ | ✗ | ✗ | **No** |
| Password reset | ✗ | ✗ | ✗ | **No** |
| 2FA / TOTP | ✗ | ✗ | ✗ | **No** |
| SSO | ✗ | ✗ | ✗ | **No** |
| Logout / token revocation | ✗ | ✗ | ✗ | **No** |
| RBAC route guards | ✗ | ✗ | ✗ | **No** |

### Dashboard

| Feature | Web | Mobile | API Backend | Complete? |
|---------|-----|--------|-------------|-----------|
| Portfolio summary (equity, PnL, Sharpe) | ✓ | ✓ | ✓ | Yes (demo data) |
| Active agents list | ✓ | ✓ | ✓ | Yes (demo data) |
| Activity feed | ✓ | ✓ | ✓ | Yes (demo data) |
| Risk snapshot | ✗ | ✓ | ✓ | Mobile only |
| Performance charts (visual) | ✗ | ✗ | ✗ | **No** |
| Real-time telemetry (SSE) | ✓ | ✓ | ✓ | Yes |
| Alerts panel | ✗ | ✓ | ✓ | Mobile only |

### Agent Management

| Feature | Web | Mobile | API Backend | Complete? |
|---------|-----|--------|-------------|-----------|
| View agent list | ✓ | ✓ | ✓ | Yes (demo data) |
| View agent detail | ✓ | ✓ | ✓ | Yes (demo data) |
| Create agent | ✗ | ✓ | ✓ | Web missing |
| Edit agent config | ✗ | ✗ | ✗ | **No** |
| Start / pause / stop agent | ✗ | ✓ | ✓ | Web missing |
| Emergency kill switch | ✗ | ✓ | ✗ | API not enforced |
| Agent AI reasoning summary | ✗ | ✓ | ✓ | Mobile only (demo) |
| Live telemetry per agent | ✓ (global) | ✓ | ✓ | Yes |
| Open positions view | ✗ | ✓ | ✓ | Web missing |
| Backtest trigger | ✗ | ✗ | ✗ | **No** |

### AI Trading Engine

| Feature | Complete? |
|---------|-----------|
| Fundamental Analyst Agent | **No** |
| Sentiment Analyst Agent | **No** |
| News Analyst Agent | **No** |
| Technical Analyst Agent | **No** |
| Parallel analyst execution | **No** |
| Research Lead consensus | **No** |
| Trader Agent (Buy/Sell/Hold) | **No** |
| Risk Manager validation | **No** |
| OpenAI GPT-4 integration | **No** |
| Live price / OHLCV data | **No** |
| News feed integration | **No** |
| Sentiment score provider | **No** |
| MACD / RSI calculation | **No** |
| Scheduled / event triggers | **No** |
| Telemetry event publishing | **No** |
| Backtesting engine | **No** |

### DEX & Blockchain

| Feature | Complete? |
|---------|-----------|
| Agent wallet creation | **No** |
| Uniswap V3 (Base + Arbitrum) | **No** |
| SushiSwap (Base + Arbitrum + BNB) | **No** |
| PancakeSwap (BNB Chain) | **No** |
| Lithosphere DEX | **No** |
| Best price router | **No** |
| Transaction signing + submission | **No** |
| Confirmation tracking + tx_hash | **No** |
| Lithosphere audit trail writes | **No** |
| QUANTT ERC-20 payment verification | **No** |

### Enterprise / B2B

| Feature | Complete? |
|---------|-----------|
| Organisation registration | **No** |
| API key generation + auth | **No** |
| Sub-user management | **No** |
| Usage metering | **No** |
| Rate limiting per API key | **No** |
| Invoice generation | **No** |
| On-chain payment verification | **No** |
| Enterprise admin web dashboard | **No** |

### Portfolio & Market Data

| Feature | Web | Mobile | API Backend | Complete? |
|---------|-----|--------|-------------|-----------|
| Portfolio summary | ✓ | ✓ | ✓ | Yes (demo) |
| Open positions detail | ✗ | ✓ | ✓ | Web missing |
| Chain-specific balances | ✗ | ✓ | ✓ | Web missing |
| Performance charts | ✗ | ✗ | ✗ | **No** |
| Drawdown history chart | ✗ | ✗ | ✗ | **No** |
| Win rate / Sharpe analytics | ✗ | ✓ | ✓ | Mobile only (demo) |
| OHLCV candlestick charts | ✗ | ✗ | ✗ | **No** |
| MACD / RSI indicators | ✗ | ✗ | ✗ | **No** |
| Live price feed | ✗ | ✗ | ✗ | **No** |
| Sentiment scores | ✗ | ✗ | ✗ | **No** |
| News feed | ✗ | ✗ | ✗ | **No** |

### Billing & Payments

| Feature | Web | Mobile | API Backend | Complete? |
|---------|-----|--------|-------------|-----------|
| Subscription plan display | ✗ | ✓ | ✗ | UI only (demo) |
| Usage cost breakdown | ✗ | ✓ | ✗ | Demo data |
| Invoice list | ✗ | ✓ | ✗ | Demo data |
| Pay invoice on-chain | ✗ | ✓ | ✗ | UI only (no chain logic) |
| Wallet balance (QUANTT token) | ✗ | ✓ | ✗ | Demo data |
| Plan upgrade self-serve | ✗ | ✗ | ✗ | **No** |

### Copilot

| Feature | Web | Mobile | API Backend | Complete? |
|---------|-----|--------|-------------|-----------|
| Chat UI | ✗ | ✓ | ✓ | Mobile only |
| Rule-based responses | — | ✓ | ✓ | Working (demo) |
| LLM-powered responses | ✗ | ✗ | ✗ | **No** |
| Control agents via chat | ✗ | ✗ | ✗ | **No** |

---

## 6. Database Schema Status

### Currently in Prisma schema (`apps/api/prisma/schema.prisma`)

```
User
  id, email, name, passwordHash, role, createdAt

RefreshToken
  id, userId, tokenHash, expiresAt, createdAt, revokedAt

TelemetryCursor
  id, source, externalId, payload, createdAt
```

### Models that NEED to be added

| Model | Purpose | Fields Needed |
|-------|---------|---------------|
| `Organisation` | Enterprise tenant | id, name, plan, createdAt |
| `OrgMember` | Sub-user within org | id, orgId, userId, role (owner/member/viewer) |
| `ApiKey` | B2B API keys | id, orgId, keyHash, label, scopes, createdAt, revokedAt |
| `ApiUsageLog` | Usage metering | id, apiKeyId, endpoint, calledAt, creditsUsed |
| `Agent` | Trading agent config | id, userId, name, strategy, chains, capital, riskLevel, maxLoss, maxPosition, autopilot, status |
| `AgentRun` | Single execution run | id, agentId, triggeredAt, status, analystResults, traderDecision, riskCheck, txHash |
| `Position` | Open position | id, agentId, symbol, chain, side, size, entryPrice, currentPrice, pnl |
| `Trade` | Completed trade | id, agentId, runId, symbol, side, sizeUsd, entryPrice, exitPrice, pnl, txHash, dex, chain |
| `AuditTrail` | Blockchain audit hashes | id, runId, lithoTxHash, payload, createdAt |
| `Invoice` | Billing invoice | id, orgId/userId, amount, currency, status, dueDate, paidAt, txHash |
| `Subscription` | Plan subscription | id, userId/orgId, plan, startedAt, endsAt, status |
| `Alert` | Platform alert | id, userId, type, severity, title, message, acknowledgedAt, createdAt |
| `DeviceToken` | Push notification | id, userId, token, platform (ios/android), createdAt |
| `EmailVerification` | Email OTP | id, userId, otp, expiresAt |
| `PasswordReset` | Reset tokens | id, userId, tokenHash, expiresAt, usedAt |

---

## 7. API Endpoint Inventory

### Implemented

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| POST | /v1/auth/register | Public | |
| POST | /v1/auth/login | Public | |
| POST | /v1/auth/refresh | Cookie | |
| POST | /v1/auth/wallet/challenge | Public | |
| POST | /v1/auth/wallet/verify | Public | |
| GET | /v1/dashboard | JWT | |
| GET | /v1/agents/:id | JWT | |
| GET | /v1/telemetry | JWT | |
| GET | /v1/telemetry/stream | JWT | SSE |
| GET | /v1/mobile/overview | JWT | |
| GET | /v1/mobile/agents/:id | JWT | |
| POST | /v1/mobile/agents | JWT | |
| POST | /v1/mobile/agents/:id/state | JWT | |
| GET | /v1/mobile/alerts | JWT | |
| POST | /v1/mobile/alerts/:id/ack | JWT | |
| GET | /v1/mobile/wallet | JWT | |
| GET | /v1/mobile/billing | JWT | |
| GET | /v1/mobile/marketplace | JWT | |
| GET | /v1/mobile/social | JWT | |
| GET | /v1/mobile/enterprise | JWT | |
| POST | /v1/mobile/wallet/pay | JWT | |
| POST | /v1/mobile/copilot | JWT | |

### Needs to be Built

| Method | Path | Priority |
|--------|------|----------|
| POST | /v1/auth/logout | High |
| POST | /v1/auth/verify-email | High |
| POST | /v1/auth/forgot-password | High |
| POST | /v1/auth/reset-password | High |
| POST | /v1/auth/2fa/setup | High |
| POST | /v1/auth/2fa/verify | High |
| GET | /v1/portfolio | High |
| GET | /v1/billing | High |
| GET | /v1/marketplace | Medium |
| GET | /v1/market/:symbol | Medium |
| GET | /v1/agents/:id/backtest | Medium |
| PATCH | /v1/agents/:id | High |
| DELETE | /v1/agents/:id | High |
| GET | /v1/admin/users | High |
| POST | /v1/admin/users | High |
| DELETE | /v1/admin/users/:id | High |
| PATCH | /v1/admin/users/:id/role | High |
| GET | /v1/admin/organizations | High |
| POST | /v1/admin/organizations | High |
| GET | /v1/admin/invoices | High |
| POST | /v1/enterprise/api-keys | High |
| DELETE | /v1/enterprise/api-keys/:id | High |
| GET | /v1/enterprise/usage | High |
| POST | /v1/notifications/register-device | High |
| POST | /v1/notifications/send | High (internal) |

---

## 8. Priority Build Order

Ordered by dependency and business impact:

### Phase 1 — Foundation & Security (Do First)

1. **RBAC middleware** — Add role guard to all protected Fastify routes; without this, any logged-in user can access admin endpoints
2. **Email verification** — OTP on registration; block login until verified
3. **Logout endpoint** — Revoke refresh token, clear cookies
4. **Password reset flow** — Forgot password → email OTP → reset
5. **Database schema expansion** — Add `Agent`, `Trade`, `Position`, `Invoice`, `Subscription`, `Organisation`, `ApiKey`, `Alert`, `DeviceToken` tables via Prisma migrations

### Phase 2 — Core AI Engine (Highest Business Value)

6. **OpenAI GPT-4 integration** — Wire API key into Python backend
7. **Market data providers** — Integrate live price, OHLCV, news, sentiment APIs
8. **Technical indicators** — MACD, RSI calculation engine
9. **4 Analyst agents** — Fundamental, Sentiment, News, Technical (parallel execution)
10. **Research Lead agent** — Consensus synthesis
11. **Trader Agent** — Buy / Sell / Hold decision with size and rationale
12. **Risk Manager** — Validate against user rules before any trade
13. **Telemetry publisher** — Emit real events to Redis `quantt.telemetry.v1`

### Phase 3 — DEX Execution

14. **Agent wallet creation** — Generate EVM wallet per agent, secure key storage
15. **Best price router** — Query Uniswap V3, SushiSwap, PancakeSwap simultaneously
16. **DEX adapters** — One adapter per DEX per chain (4 DEXes × chains)
17. **Transaction signing & submission** — Sign with agent wallet, submit to RPC
18. **Confirmation tracking** — Poll for tx receipt, store `tx_hash`
19. **Lithosphere audit trail** — Hash and write every AI decision on-chain

### Phase 4 — Web App Completion

20. **Create Agent page** — Full form matching mobile's `agent/create.tsx`
21. **Edit Agent page** — Load config, patch via API
22. **Agent controls on web** — Start / Pause / Stop / Kill switch buttons
23. **Portfolio page** — Open positions, chain breakdown, performance charts
24. **Billing page** — Plan, invoices, QUANTT payment flow
25. **Admin dashboard** — User management, org management, API keys, invoices
26. **Performance charting** — Integrate Recharts or TradingView Lightweight Charts
27. **OHLCV candlestick charts** — Market data tab on web

### Phase 5 — Enterprise & B2B

28. **Organisation registration flow** — B2B sign-up
29. **API key generation & auth** — SHA-256 keys, middleware to validate on inbound
30. **Sub-user management** — Owner / Member / Viewer roles within org
31. **Usage metering** — Redis token bucket + DB log per API call
32. **Invoice generation** — Auto-generate at billing cycle end
33. **On-chain payment verification** — Watch ERC-20 transfer events, reconcile invoices

### Phase 6 — Notifications, Copilot & Polish

34. **Push notification backend** — APNs / FCM integration; send alerts from API
35. **Email service** — Transactional emails (verify, reset, invoice, alerts)
36. **Real Copilot** — LLM-powered responses connected to user's portfolio context
37. **2FA / TOTP** — Authenticator app setup + verify flow
38. **Backtesting engine** — Historical simulation for Pro/Enterprise
39. **Monitoring / observability** — Prometheus metrics, Sentry error tracking

---

*This document reflects the state of the codebase as of 2026-05-19. Update it as features are completed.*
