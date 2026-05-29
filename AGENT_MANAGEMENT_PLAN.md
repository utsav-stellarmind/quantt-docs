# Agent Management — Implementation Plan

> **References:** `IMPLEMENTATION_STATUS.md` §4.2 Web App Gaps, §4.4 API Gateway Gaps, §6 Database Schema Status, §8 Phase 4

---

## Overview

This document describes **exactly** what needs to be built and how to connect it, covering:

1. Prisma schema addition (`Agent` model)
2. API routes (CRUD + state change)
3. Web pages — List, Create, Detail (with controls), Edit

All four features are end-to-end dynamic (database → API → UI). No mock data after implementation.

---

## 1. Database — Prisma Schema

### Add `Agent` model to `apps/api/prisma/schema.prisma`

```prisma
model Agent {
  id           String    @id @default(cuid())
  userId       String
  name         String
  strategy     String    // momentum | mean_reversion | arbitrage | trend_following | hedging | fundamental | technical
  chains       String[]  // arbitrum | base | lithosphere | bnb
  tokens       String[]
  capitalUsd   Float     @default(0)
  stopLoss     Float     @default(5)
  takeProfit   Float     @default(10)
  maxDailyLoss Float     @default(3.5)
  status       String    @default("idle")  // active | paused | idle | learning | degraded
  autopilot    Boolean   @default(true)
  pnl30dUsd    Float     @default(0)
  pnl30dPct    Float     @default(0)
  confidence   Float?
  totalTrades  Int       @default(0)
  winRate      Float     @default(0)
  lastRunAt    DateTime?
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
  user         User      @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

### Also add the reverse relation on `User`:

```prisma
model User {
  // ... existing fields ...
  agents  Agent[]   // ADD THIS LINE
}
```

### Migration command:
```bash
npx prisma migrate dev --name add_agents --schema apps/api/prisma/schema.prisma
```

---

## 2. API Routes

### New file: `apps/api/src/routes/agents.ts`

#### Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/v1/agents` | JWT | List all agents owned by the current user |
| `POST` | `/v1/agents` | JWT | Create a new agent (status defaults to `idle`) |
| `GET` | `/v1/agents/:id` | JWT | Get single agent (owner check enforced) |
| `PATCH` | `/v1/agents/:id` | JWT | Update agent config (name, strategy, chains, risk params) |
| `POST` | `/v1/agents/:id/state` | JWT | Change agent status: `active` / `paused` / `idle` |
| `DELETE` | `/v1/agents/:id` | JWT | Delete agent (owner check enforced) |

#### Validation schemas

**POST /v1/agents — createSchema**
```ts
z.object({
  name:         z.string().min(2).max(64),
  strategy:     z.enum(['momentum','mean_reversion','arbitrage','trend_following','hedging','fundamental','technical']),
  chains:       z.array(z.enum(['arbitrum','base','lithosphere','bnb'])).min(1),
  tokens:       z.array(z.string()).default([]),
  capitalUsd:   z.number().positive(),
  stopLoss:     z.number().min(0).max(100).default(5),
  takeProfit:   z.number().min(0).max(500).default(10),
  maxDailyLoss: z.number().min(0).max(100).default(3.5),
  autopilot:    z.boolean().default(true),
})
```

**PATCH /v1/agents/:id** — same schema, all fields optional (`.partial()`)

**POST /v1/agents/:id/state**
```ts
z.object({ status: z.enum(['active','paused','idle']) })
```

#### Implementation notes

- All routes use `preHandler: [app.authenticate]`
- Owner enforcement: `if (!agent || agent.userId !== user.id) throw app.httpErrors.notFound(...)`
- `POST /state` with `active` should also set `lastRunAt: new Date()`
- `DELETE` returns `204 No Content`
- Rate limit on `POST /v1/agents`: `{ max: 20, timeWindow: '1 minute' }`

#### Registration in `apps/api/src/index.ts`

```ts
// Add import:
import { agentRoutes } from "./routes/agents.js";

// Add registration (after mobileRoutes):
app.register(agentRoutes);
```

---

## 3. Web Frontend Helper — `apps/web/lib/auth.ts`

Add `apiPatch` alongside the existing helpers:

```ts
export async function apiPatch(path: string, body: unknown): Promise<Response> {
  return fetch(`${API_BASE}${path}`, {
    method: "PATCH",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(body),
    credentials: "include",
  });
}
```

---

## 4. Type Extension — `apps/web/lib/types.ts`

Extend the existing `Agent` interface with DB-stored config fields:

```ts
export interface Agent {
  // existing fields unchanged...
  ownerId?: string;        // make optional (API uses userId internally)

  // config fields returned from DB
  stopLoss?:     number;
  takeProfit?:   number;
  maxDailyLoss?: number;
  autopilot?:    boolean;
  createdAt?:    string;
  updatedAt?:    string;
}
```

---

## 5. Web Pages

### 5.1 Agents List — `apps/web/app/agents/page.tsx`

**What changes:** Replace `AGENTS` mock import with real API fetch.

**Logic:**
```
useEffect → apiGet('/v1/agents') → setAgents(data)
```

**UI additions over current page:**
- Loading skeleton (3 ghost cards while fetching)
- Empty state with "No agents yet" + Create CTA when `agents.length === 0`
- **Start button** on each card (if `status === 'paused' || 'idle'`) → `apiPost('/v1/agents/:id/state', { status: 'active' })`
- **Pause button** on each card (if `status === 'active' || 'learning' || 'degraded'`) → same API with `paused`
- **Edit button** → `router.push('/agents/:id/edit')`
- **Delete button** → confirm then `apiDelete('/v1/agents/:id')` → remove from state
- Refresh button → re-call `load()`
- All action buttons optimistically update local state on success

**State machine for buttons:**
```
idle    → [Start]
paused  → [Start]
active  → [Pause] [Stop→idle]
learning → [Pause] [Stop→idle]
degraded → [Pause] [Stop→idle]
```

---

### 5.2 Create Agent — `apps/web/app/agents/create/page.tsx`

**What changes:** 4-step form submits to real API instead of `setTimeout`.

**4-step wizard:**

| Step | Fields | Validation |
|------|--------|------------|
| 1 — Basics | Name (min 2), Chain (multi-select, min 1), Capital (positive number) | All required |
| 2 — Strategy | Strategy (single select from 7 options) | Required |
| 3 — Risk | Stop Loss %, Take Profit %, Max Daily Loss %, Autopilot toggle | All positive |
| 4 — Review | Read-only summary of all fields | — |

**Submit (Step 4 → Deploy):**
```ts
const res = await apiPost('/v1/agents', {
  name, strategy, chains, tokens: [],
  capitalUsd: Number(capital),
  stopLoss: Number(stopLoss),
  takeProfit: Number(takeProfit),
  maxDailyLoss: Number(maxDailyLoss),
  autopilot,
});
// on 201: show success screen → redirect to /agents/:id
// on error: show inline error message
```

**States:** `idle | submitting | success | error`

**Continue button** is disabled if current step validation fails.

**Deploy note in Review:** Agent starts in `idle` state — user must manually start it.

---

### 5.3 Agent Detail — `apps/web/app/agents/[id]/page.tsx`

**What changes:** Replace `AGENTS.find()` with real API fetch + add full control bar.

**Data fetch:**
```ts
useEffect → apiGet('/v1/agents/:id') → setAgent(data)
// if 404 → router.push('/agents')
```

**Control bar (top-right, always visible):**

| Button | When Shown | API Call |
|--------|-----------|---------|
| Start | `status === 'idle' \|\| 'paused'` | `POST /v1/agents/:id/state { status: 'active' }` |
| Pause | `status === 'active' \|\| 'learning' \|\| 'degraded'` | `POST /v1/agents/:id/state { status: 'paused' }` |
| Stop | `status !== 'idle'` | `POST /v1/agents/:id/state { status: 'idle' }` |
| Edit | Always | Link to `/agents/:id/edit` |
| Delete | Always | Confirm dialog → `DELETE /v1/agents/:id` → redirect to `/agents` |
| Refresh | Always | Re-fetch agent |

**Flash messages:** After each action show a 3-second inline toast (success = green, error = red).

**Tabs:**

| Tab | Content |
|-----|---------|
| Overview | Two-column: Performance metrics (30d PnL, return %, win rate, total trades) + Risk Controls (stop loss, take profit, max daily loss, autopilot) |
| Config | All agent fields in a grid (ID, name, strategy, chains, capital, risk params, created date) + Edit Config link |
| Activity | Placeholder: "Trade activity will appear here once the agent starts executing." |
| AI Reasoning | Placeholder: "Analyst and trader agent reasoning will appear here after the first run." |

**KPI header row (always visible above tabs):**

```
Capital | 30d PnL | Win Rate | Total Trades | AI Confidence | Last Run
```

---

### 5.4 Edit Agent — `apps/web/app/agents/[id]/edit/page.tsx` *(new file)*

**Route:** `/agents/[id]/edit`

**Data fetch on mount:**
```ts
apiGet('/v1/agents/:id') → populate form with existing values
```

**Form sections:**

| Section | Fields |
|---------|--------|
| Basic Info | Name (text), Capital (number) |
| Trading Strategy | Strategy (button grid, single select) |
| Blockchain Networks | Chains (button grid, multi-select) |
| Risk Parameters | Stop Loss %, Take Profit %, Max Daily Loss %, Autopilot toggle |

**Save:**
```ts
const res = await apiPatch('/v1/agents/:id', { name, strategy, chains, capitalUsd, stopLoss, takeProfit, maxDailyLoss, autopilot });
// on 200: show "Changes saved!" → redirect to /agents/:id
// on error: show inline error message
```

**Client-side validation before submit:**
- Name not empty
- At least 1 chain selected
- Strategy selected
- Capital > 0

**Cancel button** → `router.push('/agents/:id')` without saving.

---

## 6. File Summary

| File | Action |
|------|--------|
| `apps/api/prisma/schema.prisma` | Add `Agent` model + `agents Agent[]` on `User` |
| `apps/api/prisma/migrations/` | Run `prisma migrate dev --name add_agents` |
| `apps/api/src/routes/agents.ts` | **Create** — 6 endpoints |
| `apps/api/src/index.ts` | Add `agentRoutes` import + register |
| `apps/web/lib/auth.ts` | Add `apiPatch` helper |
| `apps/web/lib/types.ts` | Extend `Agent` interface with config fields |
| `apps/web/app/agents/page.tsx` | Rewrite — dynamic list, inline controls |
| `apps/web/app/agents/create/page.tsx` | Rewrite — 4-step wizard with real API submit |
| `apps/web/app/agents/[id]/page.tsx` | Rewrite — real API fetch + control bar + tabs |
| `apps/web/app/agents/[id]/edit/page.tsx` | **Create** — edit form with `PATCH` |

---

## 7. Build Order

1. Prisma schema → run migration
2. `apps/api/src/routes/agents.ts` + register in `index.ts`
3. `apps/web/lib/auth.ts` (add `apiPatch`)
4. `apps/web/lib/types.ts` (extend `Agent`)
5. `apps/web/app/agents/page.tsx` (list + inline controls)
6. `apps/web/app/agents/create/page.tsx` (create wizard)
7. `apps/web/app/agents/[id]/page.tsx` (detail + control bar)
8. `apps/web/app/agents/[id]/edit/page.tsx` (edit form)

---

*This document references `IMPLEMENTATION_STATUS.md` §4.2 (Create Agent, Edit Agent, agent controls), §4.4 (PATCH/DELETE /v1/agents), §6 (Agent model fields), and §8 Phase 4.*
