# QUANTT Platform — POC Web Design Document
### Static Data · Attractive UI · Full RBAC · All Features

> **Purpose:** This document is the single source of truth for building the QUANTT web POC.
> Every screen, every component, every static data payload, and every RBAC rule is defined here.
> Developers and designers can implement directly from this document without any other reference.

---

## Table of Contents

1. [Design System](#1-design-system)
2. [RBAC — Roles & Permissions](#2-rbac--roles--permissions)
3. [Navigation Architecture](#3-navigation-architecture)
4. [Screen-by-Screen Design](#4-screen-by-screen-design)
   - 4.01 [Landing / Marketing Page](#401-landing--marketing-page)
   - 4.02 [Login Page](#402-login-page)
   - 4.03 [Register Page](#403-register-page)
   - 4.04 [Wallet Connect Page](#404-wallet-connect-page)
   - 4.05 [Dashboard — Retail Trader](#405-dashboard--retail-trader)
   - 4.06 [Dashboard — Pro / Operator](#406-dashboard--pro--operator)
   - 4.07 [Dashboard — Enterprise](#407-dashboard--enterprise)
   - 4.08 [Dashboard — Admin](#408-dashboard--admin)
   - 4.09 [Agent List Page](#409-agent-list-page)
   - 4.10 [Create Agent Page](#410-create-agent-page)
   - 4.11 [Agent Detail Page](#411-agent-detail-page)
   - 4.12 [Live Telemetry Page](#412-live-telemetry-page)
   - 4.13 [Portfolio Page](#413-portfolio-page)
   - 4.14 [Market Data Page](#414-market-data-page)
   - 4.15 [Insights & Analytics Page](#415-insights--analytics-page)
   - 4.16 [Copilot Page](#416-copilot-page)
   - 4.17 [Marketplace Page](#417-marketplace-page)
   - 4.18 [Wallet Page](#418-wallet-page)
   - 4.19 [Billing Page](#419-billing-page)
   - 4.20 [Enterprise Fleet Page](#420-enterprise-fleet-page)
   - 4.21 [Admin — User Management](#421-admin--user-management)
   - 4.22 [Admin — Organisation Management](#422-admin--organisation-management)
   - 4.23 [Admin — API Key Management](#423-admin--api-key-management)
   - 4.24 [Settings Page](#424-settings-page)
   - 4.25 [Alerts Page](#425-alerts-page)
5. [Reusable Component Library](#5-reusable-component-library)
6. [Static Data Payloads](#6-static-data-payloads)
7. [Animation & Interaction Spec](#7-animation--interaction-spec)
8. [Responsive Breakpoints](#8-responsive-breakpoints)

---

## 1. Design System

### 1.1 Color Palette

```
Background
  --bg-base        : #04080f   /* deep space black — page background */
  --bg-panel       : #0a1628   /* navy — cards and panels */
  --bg-panel-alt   : #0d1f3c   /* slightly lighter navy — nested panels */
  --bg-surface     : #112244   /* surface for inputs and tables */
  --bg-hover       : #162d55   /* row / item hover state */

Primary Accent
  --primary        : #00d4aa   /* teal — CTA buttons, active indicators, links */
  --primary-dim    : #00a882   /* teal hover */
  --primary-glow   : rgba(0,212,170,0.15) /* teal glow for cards */

Text
  --text-primary   : #e8f4ff   /* near-white — headings and important values */
  --text-secondary : #8ba7c7   /* muted blue-grey — labels and metadata */
  --text-muted     : #4a6580   /* dim — placeholders and disabled */

Status Colors
  --success        : #00e676   /* bright green */
  --success-dim    : rgba(0,230,118,0.12)
  --warning        : #ffab00   /* amber */
  --warning-dim    : rgba(255,171,0,0.12)
  --danger         : #ff4d6a   /* red-pink */
  --danger-dim     : rgba(255,77,106,0.12)
  --info           : #40c4ff   /* sky blue */
  --info-dim       : rgba(64,196,255,0.12)
  --purple         : #b388ff   /* purple — for AI/ML indicators */
  --purple-dim     : rgba(179,136,255,0.12)

Borders
  --border         : rgba(255,255,255,0.07)
  --border-active  : rgba(0,212,170,0.4)
```

### 1.2 Typography

```
Font Family  : "Inter", system-ui, sans-serif
Code Font    : "JetBrains Mono", monospace

Scale
  --text-display : 40px / 700 / -1.5px letter-spacing
  --text-h1      : 30px / 700 / -1px
  --text-h2      : 22px / 600 / -0.5px
  --text-h3      : 18px / 600 / 0px
  --text-h4      : 15px / 600 / 0px
  --text-body    : 14px / 400 / 0px
  --text-small   : 12px / 400 / 0.2px
  --text-micro   : 11px / 500 / 0.5px  /* badges, chip labels */
```

### 1.3 Spacing

```
--space-1  : 4px
--space-2  : 8px
--space-3  : 12px
--space-4  : 16px
--space-5  : 20px
--space-6  : 24px
--space-8  : 32px
--space-10 : 40px
--space-12 : 48px
--space-16 : 64px
```

### 1.4 Border Radius

```
--radius-sm   : 8px   /* chips, badges */
--radius-md   : 12px  /* inputs, small cards */
--radius-lg   : 16px  /* main cards */
--radius-xl   : 24px  /* panels */
--radius-pill : 999px /* pills, toggles */
```

### 1.5 Shadows & Glows

```
Card shadow   : 0 4px 24px rgba(0,0,0,0.4)
Primary glow  : 0 0 32px rgba(0,212,170,0.18)
Danger glow   : 0 0 24px rgba(255,77,106,0.2)
Inset border  : inset 0 1px 0 rgba(255,255,255,0.06)
```

### 1.6 Glassmorphism Effect (for hero cards)

```css
background    : rgba(10, 22, 40, 0.7);
backdrop-filter: blur(20px);
border        : 1px solid rgba(255,255,255,0.08);
box-shadow    : 0 8px 32px rgba(0,0,0,0.5);
```

### 1.7 Icon Library

Use **Lucide React** icons throughout. Key icons:

```
TrendingUp, TrendingDown, Activity, Zap, Shield, Brain,
Bot, BarChart3, PieChart, Layers, RefreshCw, Play, Pause, Square,
AlertTriangle, Bell, Wallet, CreditCard, Users, Settings,
ChevronRight, ChevronDown, ArrowUpRight, ArrowDownRight,
Globe, Link2, Copy, Eye, EyeOff, LogOut, Plus, Trash2,
CheckCircle, XCircle, Clock, Star, Award, Cpu, Server
```

---

## 2. RBAC — Roles & Permissions

### 2.1 Role Definitions

```
ADMIN           → Full platform control
PRO_OPERATOR    → Advanced trader — unlimited agents, fleet view, analytics
RETAIL_TRADER   → Basic — max 2 agents, own data only
ENTERPRISE      → Org-level — sub-users, API keys, fleet management
ENTERPRISE_MEMBER → Can create agents and trade within org
ENTERPRISE_VIEWER  → Read-only within org
```

### 2.2 Route Guard Matrix

| Route | ADMIN | PRO_OPERATOR | RETAIL_TRADER | ENTERPRISE | ENT_MEMBER | ENT_VIEWER |
|-------|-------|-------------|--------------|------------|------------|------------|
| `/dashboard` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `/agents` | ✓ | ✓ | ✓ (max 2) | ✓ | ✓ | ✓ (read) |
| `/agents/create` | ✓ | ✓ | ✓ (if <2) | ✓ | ✓ | ✗ |
| `/portfolio` | ✓ | ✓ (own) | ✓ (own) | ✓ (org) | ✓ (own) | ✓ (org, read) |
| `/market` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `/insights` | ✓ | ✓ | Limited | ✓ | ✓ | ✓ |
| `/live` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `/copilot` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `/marketplace` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `/wallet` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `/billing` | ✓ (all) | ✓ (own) | ✓ (own) | ✓ (org) | ✗ | ✗ |
| `/alerts` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `/enterprise` | ✓ | ✗ | ✗ | ✓ | ✗ | ✗ |
| `/admin/*` | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| `/settings` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

### 2.3 Feature-Level Guards

```
Create Agent button    → Hidden for ENT_VIEWER; disabled (with tooltip) for RETAIL_TRADER at 2-agent limit
Edit Agent button      → Hidden for ENT_VIEWER
Kill Switch            → Available to ADMIN, PRO_OPERATOR, ENTERPRISE, ENT_MEMBER (own agents)
Admin nav link         → Visible only to ADMIN
Enterprise nav section → Visible to ENTERPRISE + ADMIN
Fleet Management       → PRO_OPERATOR, ENTERPRISE, ADMIN
Insights (full)        → PRO_OPERATOR, ENTERPRISE, ADMIN
Insights (basic)       → RETAIL_TRADER (win rate only, no strategy breakdown)
API Keys               → ENTERPRISE (own org) + ADMIN (all orgs)
User Management        → ADMIN only
Role Assignment        → ADMIN only
Billing (all users)    → ADMIN only
```

### 2.4 POC Static Role Switcher

Add a **Dev Role Switcher** pill in the top-right corner (only in POC mode):

```
┌─────────────────────────────────────────────────┐
│  👤 Viewing as: [ Admin ▼ ]   [POC MODE BADGE]  │
└─────────────────────────────────────────────────┘

Dropdown options:
  • Admin
  • Pro / Operator
  • Retail Trader
  • Enterprise Owner
  • Enterprise Member
  • Enterprise Viewer
```

Switching role re-renders the entire app with the correct navigation, feature flags, and data scope — no page reload needed.

---

## 3. Navigation Architecture

### 3.1 Top Navigation Bar

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  [QUANTT logo + wordmark]     [Search ⌘K]    [🔔 3]  [Role Switcher]  [Avatar] │
└────────────────────────────────────────────────────────────────────────────────┘

Logo: "Q" in teal gradient square + "QUANTT" wordmark
Search: Command palette (⌘K) — search agents, tokens, pages
Bell: Alert count badge in danger red
Avatar: User avatar with dropdown (Profile, Settings, Logout)
```

### 3.2 Left Sidebar Navigation

```
Width: 240px (collapsed: 64px icon-only)

─── MAIN ────────────────────────
  🏠  Dashboard
  🤖  Agents
  📡  Live Feed
  📊  Portfolio

─── TRADING ─────────────────────
  📈  Market Data
  💡  Insights          [PRO+ badge on RETAIL]
  🧠  Copilot
  🛒  Marketplace

─── FINANCE ─────────────────────
  👛  Wallet
  💳  Billing

─── ENTERPRISE ──────────────────  [shown to ENTERPRISE + ADMIN only]
  🏭  Fleet Management
  🔑  API Keys

─── ADMIN ───────────────────────  [shown to ADMIN only]
  👥  Users
  🏢  Organisations
  🧾  All Invoices

─── SYSTEM ──────────────────────
  🔔  Alerts
  ⚙️  Settings

────────────────────────────────
  Kill Switch  🔴 [EMERGENCY STOP ALL]
```

### 3.3 Kill Switch Component

```
Located at the bottom of sidebar, always visible for eligible roles.

Normal state:
┌──────────────────────────────────┐
│  ⏹  Emergency Stop All Agents   │  ← grey border, subtle
└──────────────────────────────────┘

Active / danger state (on hover):
┌──────────────────────────────────┐
│  ⏹  Emergency Stop All Agents   │  ← danger red border + glow
└──────────────────────────────────┘

Confirmation modal appears on click before executing.
```

---

## 4. Screen-by-Screen Design

---

### 4.01 Landing / Marketing Page

**Route:** `/`  **Auth Required:** No

#### Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  NAVBAR: Logo  |  Features  Product  Pricing  Docs  |  Login  Sign Up │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   HERO SECTION                                                   │
│   ─────────────────────────────────────────────────             │
│   Eyebrow: "AI-Powered Autonomous Trading"                       │
│                                                                  │
│   Headline (60px bold gradient):                                 │
│   "Let AI Trade                                                  │
│    While You Sleep."                                             │
│                                                                  │
│   Subheadline (18px muted):                                      │
│   "Deploy intelligent trading agents that research, decide,      │
│    and execute — 24/7 across Ethereum, BNB Chain, Arbitrum,      │
│    and Lithosphere — with every decision recorded on-chain."     │
│                                                                  │
│   [  Get Started Free  ]  [  Watch Demo →  ]                    │
│                                                                  │
│   ── Stats Bar ──────────────────────────────────────────────── │
│   $2.4B Traded   |  14,200 Active Agents  |  99.8% Uptime       │
│                                                                  │
│   ── Live Mini Dashboard Preview (animated) ────────────────── │
│   A glass morphism card showing a live portfolio graph + 3      │
│   agent cards scrolling by with status + PnL indicators         │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  HOW IT WORKS  (6-step horizontal flow with icons)               │
│  Create → Analyse → Research → Decide → Risk Check → Execute    │
├─────────────────────────────────────────────────────────────────┤
│  FEATURES  (3-column grid of feature cards)                      │
│  [4 Parallel AI Analysts]  [On-Chain Audit Trail]  [Best DEX Price Router] │
│  [Real-Time Telemetry]     [Multi-Chain Support]   [Risk Manager]          │
├─────────────────────────────────────────────────────────────────┤
│  ROLES — 4 cards side by side                                    │
│  Retail Trader | Pro Operator | Enterprise | Admin               │
├─────────────────────────────────────────────────────────────────┤
│  SUPPORTED CHAINS & DEXes (logo strip)                           │
│  Base · Arbitrum · BNB Chain · Lithosphere                       │
│  Uniswap · SushiSwap · PancakeSwap · Lithosphere DEX            │
├─────────────────────────────────────────────────────────────────┤
│  TESTIMONIALS (3 cards)                                          │
├─────────────────────────────────────────────────────────────────┤
│  CTA BANNER: "Start trading smarter today."  [Sign Up Free]     │
├─────────────────────────────────────────────────────────────────┤
│  FOOTER                                                          │
└─────────────────────────────────────────────────────────────────┘
```

**Headline gradient:** `linear-gradient(135deg, #00d4aa 0%, #40c4ff 50%, #b388ff 100%)`

---

### 4.02 Login Page

**Route:** `/login`  **Auth Required:** No

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│   LEFT PANEL (50%)  — Brand & animated art                       │
│   ─────────────────────────────────────────                      │
│   QUANTT logo                                                     │
│   "The Future of Autonomous Trading"                              │
│                                                                   │
│   Animated floating cards showing:                                │
│   • "ETH/USDC — BUY $2,000 — +2.4% ✓"                           │
│   • "Risk Check — PASSED ✓"                                       │
│   • "Best Price — SushiSwap — $1,995"                            │
│   (cards float upward with CSS animation, staggered)             │
│                                                                   │
│   RIGHT PANEL (50%) — Auth form                                  │
│   ─────────────────────────────────────────                      │
│   "Welcome Back"  (h1)                                           │
│   "Sign in to your QUANTT account"  (muted)                      │
│                                                                   │
│   TAB GROUP:  [ Email ]  [ Wallet ]                              │
│                                                                   │
│   EMAIL TAB:                                                      │
│   ┌─ Email address ─────────────────────────────────────┐        │
│   │  trader@example.com                                  │        │
│   └──────────────────────────────────────────────────────┘       │
│   ┌─ Password ───────────────────────────────────────────┐       │
│   │  ••••••••••••  [👁 show]                             │        │
│   └──────────────────────────────────────────────────────┘       │
│   [ Forgot password? ]                     (right aligned)       │
│   [ Sign In  →  ]              (full width teal gradient button) │
│                                                                   │
│   WALLET TAB:                                                     │
│   ┌──────────────────────────────────────────────────────┐       │
│   │  [MetaMask icon]  Connect MetaMask                   │        │
│   └──────────────────────────────────────────────────────┘       │
│   ┌──────────────────────────────────────────────────────┐       │
│   │  [Litho icon]  Connect Lithosphere Wallet            │        │
│   └──────────────────────────────────────────────────────┘       │
│   Signed message preview shown after connect                     │
│                                                                   │
│   Divider: ── or ──                                              │
│   "Don't have an account?  [Create one →]"                       │
│                                                                   │
│   POC QUICK LOGIN SHORTCUTS:                                      │
│   ┌────────────────────────────────────────────────────┐         │
│   │  🚀 Quick Login (POC)                              │         │
│   │  [Admin] [Pro Trader] [Retail] [Enterprise]        │         │
│   └────────────────────────────────────────────────────┘         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Static POC credentials shown:**
```
Admin:      admin@quantt.io      / demo1234!@#$
Pro:        pro@quantt.io        / demo1234!@#$
Retail:     retail@quantt.io     / demo1234!@#$
Enterprise: enterprise@quantt.io / demo1234!@#$
```

---

### 4.03 Register Page

**Route:** `/register`  **Auth Required:** No

```
Same split-panel layout as login.

Form fields:
  Full Name      : [text input]
  Email address  : [email input]
  Password       : [password + strength meter]
  Confirm Password: [password]
  Role intent    : [Retail Trader ▼]  (dropdown — final role set by admin)

Password strength meter:
  Weak   → 1 bar red
  Fair   → 2 bars amber
  Strong → 3 bars teal
  Great  → 4 bars green + checkmark

Terms checkbox: "I agree to the Terms of Service and Privacy Policy"

[Create Account →]

Already have one? Sign In
```

---

### 4.04 Wallet Connect Page

**Route:** `/connect-wallet`  **Auth Required:** JWT (no wallet yet)

```
Centered card (600px wide):

  "Connect Your Wallet"
  "Link an EVM wallet to enable on-chain trading and payments."

  ┌────────────────────────────────────────────┐
  │  [🦊 MetaMask]                             │  ← hover: teal border glow
  │  Connect using browser extension           │
  └────────────────────────────────────────────┘

  ┌────────────────────────────────────────────┐
  │  [⛓ Lithosphere Wallet]                   │
  │  Native LITHO chain wallet                 │
  └────────────────────────────────────────────┘

  After connect — shows address + chain:
  ┌────────────────────────────────────────────┐
  │  ✓  0x4f3a...8b2c  │  Chain: Lithosphere  │
  │  [Sign Message to Verify]                  │
  └────────────────────────────────────────────┘

  [Skip for now] (text link, small)
```

---

### 4.05 Dashboard — Retail Trader

**Route:** `/dashboard`  **Role:** RETAIL_TRADER

```
PAGE HEADER:
  "Good morning, Alex 👋"
  "Your 2 agents are active. Portfolio is up 3.2% today."

─── ROW 1: KPI CARDS (4 cards, equal width) ─────────────────────

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  Total Equity    │ │  Today's PnL     │ │  Active Agents   │ │  30d Sharpe      │
│                  │ │                  │ │                  │ │                  │
│  $12,847.32      │ │  +$411.20        │ │      2 / 2       │ │      1.84        │
│  ↑ +3.21%        │ │  ↑ +3.31%        │ │  [limit reached] │ │  above avg       │
│  vs yesterday    │ │  since midnight  │ │  [Upgrade ↗]     │ │                  │
└──────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘

Card design: dark navy bg, teal left-border accent, subtle number counter animation on load.
"Upgrade ↗" link on agent count card if at limit — links to /billing.

─── ROW 2: PORTFOLIO CHART + RISK SNAPSHOT ──────────────────────

LEFT (65%):
┌──────────────────────────────────────────────────────────────┐
│  Portfolio Performance                [1D] [1W] [1M] [3M]   │
│                                                              │
│  $12,847                                              ╭╮     │
│  ──────────────────────────────────────────────╭─────╯  │   │
│  $11,000 ─────────────────────────────────────╯         │   │
│  $10,000                                                │   │
│         Jan    Feb    Mar    Apr    May                  │   │
│                                                              │
│  Area chart: teal gradient fill (#00d4aa → transparent)      │
└──────────────────────────────────────────────────────────────┘

RIGHT (35%):
┌──────────────────────────────────┐
│  Risk Snapshot                   │
│                                  │
│  Risk Level:   ●●●○○  MODERATE  │
│  Exposure:     68% of portfolio  │
│  Daily Loss:   -$124.50          │
│  Max Drawdown: -4.2%             │
│  Kill Switch:  [OFF ○────] [ON]  │
│                                  │
│  ── Limits ──────────────────── │
│  Max Position:  $2,500           │
│  Max Daily Loss: $800            │
│  Current Loss:   $124.50 ✓       │
└──────────────────────────────────┘

─── ROW 3: MY AGENTS ────────────────────────────────────────────

"My Agents" header + [+ Create Agent] button (greyed + tooltip "Limit reached — upgrade to Pro")

┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Agent Name    │ Strategy        │ Chain    │ Status    │ PnL 30d   │ Actions        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  🤖 AlphaBot  │ Momentum        │ Arbitrum │ ● Active  │ +$842.10  │ [Pause] [View] │
│  🤖 EthShield │ Mean Reversion  │ Base     │ ● Active  │ +$229.80  │ [Pause] [View] │
└─────────────────────────────────────────────────────────────────────────────────────┘

─── ROW 4: ACTIVITY FEED ────────────────────────────────────────

"Recent Activity" header + live dot (●) + "Live"

┌──────────────────────────────────────────────────────────────────────┐
│  ● TRADE    AlphaBot bought ETH/USDC $1,800  +2.1%   2 mins ago     │
│  ◉ RISK     EthShield: daily loss limit 82% reached  5 mins ago      │
│  ● TRADE    AlphaBot: Research Lead — BUY decision   8 mins ago      │
│  ● SYSTEM   AlphaBot: scheduled run triggered        12 mins ago     │
│  ● TRADE    EthShield sold WBTC/USDC $500  -0.3%    18 mins ago     │
└──────────────────────────────────────────────────────────────────────┘

Colour coding: trade=teal dot, risk=amber dot, system=blue dot, error=red dot
Each row is clickable → opens agent detail at that event
```

---

### 4.06 Dashboard — Pro / Operator

**Route:** `/dashboard`  **Role:** PRO_OPERATOR

Same as Retail but with these additions:

```
─── ROW 1: KPI CARDS (6 cards) ─────────────────────────────────

  Total Equity | Today PnL | Active Agents | 30d Sharpe | Win Rate | Model Confidence
  $284,720     | +$9,241   | 12 agents     | 2.31       | 67.4%    | 84%

─── EXTRA SECTION: FLEET OVERVIEW ──────────────────────────────

┌──────────────────────────────────────────────────────────────────────┐
│  Fleet Health  ●●●●●●●●●○  91%                                      │
│                                                                      │
│  12 agents total:  9 Active  │  2 Paused  │  1 Degraded             │
│                                                                      │
│  [View All Agents →]  [Fleet Management →]                          │
└──────────────────────────────────────────────────────────────────────┘

─── AGENT TABLE: shows all 12 agents, paginated ─────────────────

─── EXTRA CHARTS: Strategy Contribution ────────────────────────

┌─────────────────────────────┐  ┌─────────────────────────────────┐
│  PnL by Strategy            │  │  Chain Distribution             │
│                             │  │                                 │
│  Momentum     45% ████████  │  │  Arbitrum  38%  ████████        │
│  Mean Rev     28% █████     │  │  Base      31%  ██████          │
│  Arbitrage    17% ███       │  │  BNB Chain 22%  ████            │
│  Hedging      10% ██        │  │  LITHO      9%  ██              │
└─────────────────────────────┘  └─────────────────────────────────┘
```

---

### 4.07 Dashboard — Enterprise

**Route:** `/dashboard`  **Role:** ENTERPRISE

```
─── ORG HEADER ──────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│  🏢 Apex Trading Corp              Plan: Enterprise Pro          │
│  12 users  │  3 API keys  │  Next invoice: Jun 1, 2026          │
└──────────────────────────────────────────────────────────────────┘

─── KPI CARDS (6) ───────────────────────────────────────────────

  Org Equity | Org PnL Today | Active Agents (org) | API Calls | Monthly Spend | Members
  $1.24M     | +$41,200      | 47 agents           | 84,210    | $12,400 QUANTT| 12

─── BOTTOM SECTIONS ─────────────────────────────────────────────

[Sub-User Activity]  [API Usage Chart]  [Invoice Status]
```

---

### 4.08 Dashboard — Admin

**Route:** `/dashboard`  **Role:** ADMIN

```
─── PLATFORM OVERVIEW HEADER ────────────────────────────────────

"Platform Command Centre"
Last updated: Live ● 09:41:32

─── KPI CARDS (8) ───────────────────────────────────────────────

  Total Users | Active Agents | Platform Equity | 24h Volume | 
  3,241       | 8,847         | $284M           | $42M       |

  Active Orgs | API Keys | Pending Invoices | System Health
  127         | 342      | 18               | ✓ 100%

─── CHARTS ──────────────────────────────────────────────────────

[New User Registrations (30d bar chart)]  [Agent Activity (line chart)]

─── TABLES ──────────────────────────────────────────────────────

[Recent Users]  [Pending Invoices]  [System Alerts]

─── QUICK ACTIONS ───────────────────────────────────────────────

[Create User]  [Create Org]  [View All Invoices]  [System Config]
```

---

### 4.09 Agent List Page

**Route:** `/agents`  **Auth Required:** Any authenticated role

```
PAGE HEADER:
  "Agents"
  "Deploy and manage your AI trading agents."
  [+ Create Agent]  ← disabled with tooltip for RETAIL at limit, hidden for ENT_VIEWER

─── FILTER & SEARCH BAR ─────────────────────────────────────────

[🔍 Search agents...]  [Status ▼]  [Chain ▼]  [Strategy ▼]  [Sort ▼]

─── AGENT CARDS GRID (3 columns) ────────────────────────────────

┌─────────────────────────────────────┐
│  🤖  AlphaBot                [●Active]│
│  ─────────────────────────────────  │
│  Strategy   : Momentum Trading      │
│  Chain      : Arbitrum              │
│  Capital    : $10,000               │
│  PnL (30d)  : +$842.10  ↑ +8.42%   │  ← green
│  Confidence : ████████░░  84%       │
│  Last run   : 2 mins ago            │
│                                     │
│  [▶ Run]  [⏸ Pause]  [👁 View]     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  🤖  EthShield              [●Active]│
│  ...                                │
│  PnL (30d)  : +$229.80  ↑ +4.59%   │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  🤖  ArbiBot             [◉Paused]  │
│  ...                                │
│  PnL (30d)  : -$84.20  ↓ -1.68%   │  ← red
└─────────────────────────────────────┘

Status badge colours:
  Active   → teal background, bright dot
  Paused   → amber background
  Learning → purple background
  Degraded → danger background, pulsing dot
  Idle     → grey background
  Error    → danger background, urgent

─── ADMIN / PRO EXTRA: TABLE VIEW TOGGLE ─────────────────────

[Cards] [Table] tab toggle in top-right

Table view columns:
  Name | Strategy | Chain | Capital | Status | PnL 30d | Confidence | Last Run | Actions
```

**Static agent list data:**

```
1. AlphaBot       — Momentum        — Arbitrum  — $10,000 — Active    — +$842.10  — 84%
2. EthShield      — Mean Reversion  — Base       — $5,000  — Active    — +$229.80  — 71%
3. ArbiBot        — Arbitrage       — Arbitrum  — $8,000  — Paused    — -$84.20   — N/A
4. LithoTrader    — Momentum        — LITHO      — $3,000  — Active    — +$412.50  — 78%
5. BNBMatic       — Trend Follow    — BNB Chain  — $6,000  — Learning  — +$0       — 62%
6. GhostArb       — Arbitrage       — Base       — $12,000 — Active    — +$1,284   — 91%
7. SafeHold       — Hedging         — Arbitrum  — $20,000 — Active    — +$320.00  — 68%
8. SpeedBot       — Momentum        — BNB Chain  — $4,000  — Degraded  — +$142.00  — 55%
9. ValueSeeker    — Fundamental     — Base       — $7,500  — Active    — +$680.00  — 76%
10. DeltaNeutral  — Hedging         — Arbitrum  — $15,000 — Paused    — +$210.00  — N/A
11. MACDRunner    — Technical       — Base       — $5,000  — Active    — +$390.00  — 81%
12. RiskOff       — Mean Reversion  — LITHO      — $2,500  — Idle      — +$0       — N/A
```

---

### 4.10 Create Agent Page

**Route:** `/agents/create`  **Auth:** Authenticated (RETAIL max 2, ENT_VIEWER blocked)

```
MULTI-STEP WIZARD — 4 steps with progress bar at top

Progress:  ①────②────③────④
           Basic  Strategy  Risk  Review

─── STEP 1: Basic Info ──────────────────────────────────────────

  Agent Name
  ┌──────────────────────────────────────────────┐
  │  e.g. AlphaBot                               │
  └──────────────────────────────────────────────┘

  Description (optional)
  ┌──────────────────────────────────────────────┐
  │  What is this agent's purpose?               │
  └──────────────────────────────────────────────┘

  Select Chains (multi-select chip group)
  [Base] [Arbitrum] [BNB Chain] [Lithosphere]
  RETAIL → max 1 chain; PRO/ENT → all chains

  Tokens to Trade (multi-select searchable)
  [ETH] [WBTC] [LITHO] [+ Add token]

  Capital Allocation
  ┌────────────────────────────────┐
  │  $ 10,000.00                   │
  └────────────────────────────────┘
  Available balance: $42,800.00

  [Next: Strategy →]

─── STEP 2: Strategy ────────────────────────────────────────────

  Strategy Type (card selector)

  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │  📈 Momentum     │  │  📉 Mean Reversion│  │  ⚡ Arbitrage    │
  │  Ride the trend  │  │  Buy dips        │  │  Price gaps      │
  │  Risk: Medium    │  │  Risk: Low-Med   │  │  Risk: Med-High  │
  └──────────────────┘  └──────────────────┘  └──────────────────┘
  ┌──────────────────┐  ┌──────────────────┐
  │  🔺 Trend Follow │  │  🛡 Hedging      │
  │  Long-term moves │  │  Protect capital │
  │  Risk: Medium    │  │  Risk: Low       │
  └──────────────────┘  └──────────────────┘

  Aggressiveness
  Conservative ○──●──────────────○ Aggressive
  [Low] [Moderate] [High] [Maximum]

  Execution Frequency
  [Every Hour ▼]   options: Every 15m / 30m / 1h / 4h / 12h / 24h / Event-driven

  Autopilot
  OFF ○──────────────○ ON (fully autonomous — no approval needed per trade)
  Info tooltip: "When ON, the agent executes trades without asking you."

  [← Back]  [Next: Risk →]

─── STEP 3: Risk Controls ───────────────────────────────────────

  Risk Level (visual selector with gauge)
  ●──●──●──○──○  MODERATE

  Max Daily Loss ($)
  ┌──────────────┐
  │  $500        │
  └──────────────┘
  "Agent will pause if daily loss exceeds this amount."

  Max Position Size ($)
  ┌──────────────┐
  │  $2,500      │
  └──────────────┘
  "No single trade will exceed this value."

  Max Drawdown (%)
  ┌──────────────┐
  │  15%         │
  └──────────────┘

  Stop Loss per Trade (%)
  ┌──────────────┐
  │  3%          │
  └──────────────┘

  Take Profit per Trade (%)
  ┌──────────────┐
  │  6%          │
  └──────────────┘

  Slippage Tolerance (%)
  ┌──────────────┐
  │  0.5%        │
  └──────────────┘

  [← Back]  [Next: Review →]

─── STEP 4: Review & Deploy ─────────────────────────────────────

  Summary card (read-only):

  ┌──────────────────────────────────────────────────────────────┐
  │  🤖  AlphaBot                                                │
  │  ─────────────────────────────────────────────────────────  │
  │  Strategy    : Momentum Trading        Capital : $10,000     │
  │  Chain       : Arbitrum               Tokens  : ETH, WBTC   │
  │  Frequency   : Every Hour             Autopilot: ON          │
  │  Risk Level  : Moderate                                      │
  │  Max Loss    : $500/day               Max Position: $2,500   │
  │  Drawdown    : 15%                    Slippage    : 0.5%     │
  └──────────────────────────────────────────────────────────────┘

  Estimated fee: 0.24 QUANTT/month

  [← Back]  [🚀 Deploy Agent]

  After deploy → success toast + redirect to /agents/[id]
```

---

### 4.11 Agent Detail Page

**Route:** `/agents/[id]`  **Auth:** Any role (ENT_VIEWER = read-only)

```
─── AGENT HEADER ────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────────┐
│  🤖 AlphaBot                    ● Active                            │
│  Momentum Trading · Arbitrum · $10,000 capital                      │
│                                                                      │
│  [▶ Run Now]  [⏸ Pause]  [⏹ Stop]  [✏ Edit]  [🗑 Delete]          │
│   ← hidden for ENT_VIEWER                                           │
└──────────────────────────────────────────────────────────────────────┘

─── KPI STRIP ───────────────────────────────────────────────────

  PnL (30d)   | Win Rate | Total Trades | Avg Trade | Confidence | Uptime
  +$842.10    | 68.4%    | 142 trades   | +$5.93    | 84%        | 99.2%
  +8.42% ↑   |          | this month   | per trade |            |

─── TABS ────────────────────────────────────────────────────────

[Overview] [Positions] [Activity] [AI Reasoning] [Config] [Live]

─── TAB: OVERVIEW ───────────────────────────────────────────────

LEFT (60%):
  PnL chart (30-day area chart, teal gradient)
  Trade volume bars (stacked bar chart below)

RIGHT (40%):
  ┌──────────────────────────────────┐
  │  Configuration                   │
  │  ─────────────────────────────  │
  │  Strategy     Momentum           │
  │  Chain        Arbitrum           │
  │  Tokens       ETH, WBTC          │
  │  Capital      $10,000            │
  │  Frequency    Every Hour         │
  │  Autopilot    ● ON               │
  │  Risk Level   Moderate           │
  │  Max Loss     $500/day           │
  │  Max Position $2,500             │
  └──────────────────────────────────┘

─── TAB: POSITIONS ──────────────────────────────────────────────

Table:
  Symbol     | Side | Size      | Entry   | Current | PnL      | DEX
  ETH/USDC   | LONG | $1,800.00 | $2,940  | $3,012  | +$72.00  | Uniswap V3
  WBTC/USDC  | LONG | $1,200.00 | $61,200 | $62,400 | +$23.50  | SushiSwap

─── TAB: ACTIVITY ───────────────────────────────────────────────

Chronological list of every agent action:

  09:42:10  TRADE EXECUTED   Bought ETH/USDC $1,800 @ $2,940  tx: 0x4f3a...
  09:41:55  RISK CHECK       PASSED — position within limits
  09:41:50  TRADER DECISION  BUY ETH $1,800 — Momentum confirmed
  09:41:45  RESEARCH LEAD    Consensus: Bullish — 3/4 analysts agree
  09:41:30  ANALYST DONE     Technical: RSI 62, MACD bullish crossover
  09:41:30  ANALYST DONE     Sentiment: 72/100 Greed
  09:41:30  ANALYST DONE     News: No negative catalysts
  09:41:30  ANALYST DONE     Fundamental: ETH strong Q1 metrics
  09:41:28  PARALLEL START   4 analysts launched
  09:41:27  RUN TRIGGERED    Scheduled hourly run

─── TAB: AI REASONING ───────────────────────────────────────────

Full analyst debate rendered as collapsible cards:

  ┌────────────────────────────────────────────────────────────────┐
  │  📊 Fundamental Analyst                        [Score: 78/100] │
  │  ─────────────────────────────────────────────────────────── │
  │  ETH network activity is up 14% MoM. Layer-2 adoption         │
  │  accelerating. Developer activity at 6-month high. Revenue     │
  │  from gas fees stable. Verdict: BULLISH.                       │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  💬 Sentiment Analyst                          [Score: 72/100] │
  │  ─────────────────────────────────────────────────────────── │
  │  Fear & Greed Index: 72 (Greed). Social volume up 8%. Positive │
  │  mentions up 12% on X. Retail interest recovering. Verdict:    │
  │  CAUTIOUSLY BULLISH.                                            │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  📰 News Analyst                               [Score: 65/100] │
  │  ─────────────────────────────────────────────────────────── │
  │  No major negative events. SEC clarity improving. ETF inflows  │
  │  continue. One minor FUD article — low credibility source.     │
  │  Verdict: NEUTRAL to POSITIVE.                                 │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  📉 Technical Analyst                          [Score: 81/100] │
  │  ─────────────────────────────────────────────────────────── │
  │  RSI: 62 — not overbought. MACD: bullish crossover confirmed   │
  │  3h ago. Price above 50 EMA. Volume increasing. Pattern:       │
  │  ascending triangle breakout. Target: $3,150. Verdict: BUY.   │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  🧠 Research Lead Consensus                   BULLISH ↑        │
  │  ─────────────────────────────────────────────────────────── │
  │  3 of 4 analysts bullish. Technical signal strongest.          │
  │  News risk low. Sentiment supportive. Fundamental solid.       │
  │  Recommended action: BUY with moderate size. Confidence: 84%  │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  🤖 Trader Agent Decision                      BUY ↑ APPROVED  │
  │  ─────────────────────────────────────────────────────────── │
  │  Action: BUY  Symbol: ETH/USDC  Size: $1,800  DEX: Uniswap    │
  │  Rationale: Strong technical breakout + fundamental support.   │
  │  Expected return: +2.4% | Stop loss: $2,852 | TP: $3,150      │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  🛡 Risk Manager                               ✓ PASSED        │
  │  ─────────────────────────────────────────────────────────── │
  │  ✓ Position size $1,800 < limit $2,500                         │
  │  ✓ Daily loss $124 < limit $500                                │
  │  ✓ Drawdown 4.2% < limit 15%                                   │
  │  ✓ Portfolio exposure 68% — within acceptable range            │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  ⛓ On-Chain Audit                             Recorded ✓       │
  │  ─────────────────────────────────────────────────────────── │
  │  Lithosphere TX: 0x8c2a4f...b91d                              │
  │  Decision hash: sha256:a3f91b...                               │
  │  Block: #4,829,441  Timestamp: 2026-05-19 09:41:56 UTC        │
  └────────────────────────────────────────────────────────────────┘

─── TAB: CONFIG ─────────────────────────────────────────────────

Editable form version of agent configuration (same as Create Step 1-3).
[Save Changes] button — disabled for ENT_VIEWER.

─── TAB: LIVE ───────────────────────────────────────────────────

Real-time step-by-step stream (see Live Telemetry section).
```

---

### 4.12 Live Telemetry Page

**Route:** `/live`  **Auth:** Any

```
PAGE HEADER:
  "Live Feed"   ● LIVE  (pulsing green dot)
  "Real-time stream of every agent decision as it happens."

─── FILTER BAR ──────────────────────────────────────────────────

[All Agents ▼]  [All Events ▼: Trade / Risk / Analysis / System]  [🔍]  [⏸ Pause]

─── LIVE EVENT STREAM ───────────────────────────────────────────

New events slide in from top with subtle entrance animation.

  ┌──────────────────────────────────────────────────────────────────────┐
  │  09:42:10  🤖 AlphaBot  ● TRADE EXECUTED                            │
  │            Bought ETH/USDC — Size: $1,800 — Price: $2,940           │
  │            DEX: Uniswap V3 (Arbitrum) — Slippage: 0.08%             │
  │            TX: 0x4f3a8b2c...  [View on Explorer ↗]                  │
  ├──────────────────────────────────────────────────────────────────────┤
  │  09:41:55  🛡 AlphaBot  ✓ RISK CHECK PASSED                         │
  │            All 4 limits within bounds. Trade approved.               │
  ├──────────────────────────────────────────────────────────────────────┤
  │  09:41:50  🤖 AlphaBot  → TRADER DECISION: BUY                      │
  │            "Buy ETH $1,800. Momentum confirmed. Confidence: 84%"     │
  ├──────────────────────────────────────────────────────────────────────┤
  │  09:41:45  🧠 AlphaBot  RESEARCH LEAD — CONSENSUS: BULLISH           │
  │            3/4 analysts bullish. Technical signal strongest.         │
  ├──────────────────────────────────────────────────────────────────────┤
  │  09:41:30  📊 AlphaBot  ANALYST RESULTS (4 parallel)                │
  │            [Fundamental 78] [Sentiment 72] [News 65] [Technical 81]  │
  ├──────────────────────────────────────────────────────────────────────┤
  │  09:41:27  ⏰ AlphaBot  SCHEDULED RUN TRIGGERED                     │
  └──────────────────────────────────────────────────────────────────────┘

Event type colours:
  TRADE    → teal left border
  RISK     → amber (pass) / red (fail)
  ANALYST  → purple left border
  RESEARCH → blue left border
  SYSTEM   → grey left border

─── SIDE PANEL: Active Agents Status ────────────────────────────

Mini cards for each active agent showing:
  Name | Status | Last Action | Current Step in pipeline
```

---

### 4.13 Portfolio Page

**Route:** `/portfolio`  **Auth:** Any

```
PAGE HEADER:
  "Portfolio"
  "Total equity across all agents and chains."

─── TOP KPI ROW ─────────────────────────────────────────────────

Total Equity | 24h PnL     | 7d PnL     | 30d PnL     | Sharpe (30d) | Max Drawdown
$12,847.32   | +$411.20    | +$1,241.00 | +$2,847.32  | 1.84         | -4.2%
             | +3.31% ↑   | +10.7% ↑  | +28.5% ↑   | Above avg    | within limit

─── PORTFOLIO CHART ─────────────────────────────────────────────

Large area chart (full width, 280px height)
  Tabs: [Equity] [PnL] [Drawdown]
  Time: [1D] [1W] [1M] [3M] [6M] [All]

─── 2-COLUMN SECTION ────────────────────────────────────────────

LEFT (50%):
┌──────────────────────────────────────────────┐
│  Open Positions (4)                          │
│  ─────────────────────────────────────────  │
│  ETH/USDC   LONG  $1,800  +$72.00  +4.0% ↑  │
│  WBTC/USDC  LONG  $1,200  +$23.50  +1.9% ↑  │
│  LITHO/USDC LONG  $500    -$8.20   -1.6% ↓  │
│  BNB/USDC   LONG  $800    +$12.00  +1.5% ↑  │
│  ─────────────────────────────────────────  │
│  Total open:  $4,300.00  unrealised: +$99.30│
└──────────────────────────────────────────────┘

RIGHT (50%):
┌──────────────────────────────────────────────┐
│  Chain Breakdown                             │
│                                             │
│  Arbitrum    $6,420   50% ████████████      │
│  Base        $3,210   25% ██████            │
│  BNB Chain   $2,140   17% ████              │
│  Lithosphere $1,077    8% ██                │
└──────────────────────────────────────────────┘

─── TRADE HISTORY TABLE ─────────────────────────────────────────

Columns: Date | Agent | Pair | Side | Size | Entry | Exit | PnL | DEX | TX

Filters: [Date range] [Agent] [Chain] [Side: All/Buy/Sell] [Outcome: All/Win/Loss]

Pagination: 25 rows per page

Example rows:
  2026-05-19 09:42  AlphaBot  ETH/USDC  BUY  $1,800  $2,940  —       open    Uniswap  0x4f3a
  2026-05-19 07:31  EthShield WBTC/USDC SELL $500    $61,200 $62,100 +$45.00 SushiSwap 0x9b2c
  2026-05-18 22:14  AlphaBot  ETH/USDC  SELL $1,800  $2,910  $2,985  +$46.50 Uniswap  0xa12d
```

---

### 4.14 Market Data Page

**Route:** `/market`  **Auth:** Any

```
PAGE HEADER:
  "Market Data"
  "Live prices, charts, and sentiment across supported tokens."

─── SEARCH + TOKEN SELECTOR ─────────────────────────────────────

[🔍 Search token... ETH, BTC, LITHO...]

─── WATCHLIST STRIP ─────────────────────────────────────────────

Horizontal scrollable ticker:
  ETH  $2,947.42  +2.41% ↑  |  BTC  $67,240  +1.82% ↑  |  LITHO  $0.82  +5.1% ↑
  BNB  $612.30   -0.41% ↓  |  SOL  $185.20  +3.21% ↑  |  ARB   $1.24  +0.9% ↑

─── MAIN CHART PANEL ────────────────────────────────────────────

LEFT (70%):
┌──────────────────────────────────────────────────────────────────┐
│  ETH / USDC                                    [1H] [4H] [1D]   │
│  $2,947.42  +$69.42 (+2.41%)   Volume: $2.4B                    │
│                                                                  │
│  OHLCV Candlestick Chart (TradingView Lightweight Charts style)  │
│                                                                  │
│  ─── Indicators ─────────────────────────────────────────────   │
│  MACD:  Signal: BUY ↑  Histogram: positive  Fast>Slow           │
│  RSI:   62.4  (neutral-bullish zone, not overbought)             │
│                                                                  │
│  MACD chart (below candles):  green bars above zero             │
│  RSI line chart (below MACD): line at 62.4                      │
└──────────────────────────────────────────────────────────────────┘

RIGHT (30%):
┌──────────────────────────────────┐
│  ETH Token Info                  │
│  ─────────────────────────────  │
│  Market Cap   : $354B            │
│  24h Volume   : $18.4B           │
│  Circulating  : 120.3M ETH       │
│  All-Time High: $4,868           │
│  ATH Distance : -39.4%           │
│                                  │
│  ─── Sentiment ──────────────── │
│  Fear & Greed : 72 (Greed)       │
│  Score visual: ●●●●●●●●░░  72%  │
│                                  │
│  Social Volume  ↑ +8% 24h        │
│  Positive Posts ↑ +12%           │
│                                  │
│  ─── Fundamental ─────────────  │
│  Developer Activity : High ↑     │
│  On-chain Tx Volume : ↑ +14%     │
│  L2 TVL             : $42B ↑     │
│                                  │
│  ─── Latest News ─────────────  │
│  • "ETF inflows hit $420M week"  │
│  • "Vitalik proposes EIP-7002"   │
│  • "L2 fees drop 80% post Dencun"│
└──────────────────────────────────┘
```

---

### 4.15 Insights & Analytics Page

**Route:** `/insights`  **Auth:** PRO_OPERATOR+ (RETAIL gets limited view)

```
─── RETAIL TRADER LIMITED VIEW ──────────────────────────────────

Shows only:
  Win Rate: 68.4%
  Total Trades: 142
  Best Performing Agent: AlphaBot

  Upgrade banner:
  ┌──────────────────────────────────────────────────────────────┐
  │  🔒 Full Insights available on Pro Plan                      │
  │  Unlock AI reasoning breakdown, strategy analytics,          │
  │  coordination graph, and fleet health.                       │
  │  [Upgrade to Pro →]                                          │
  └──────────────────────────────────────────────────────────────┘

─── PRO+ FULL VIEW ──────────────────────────────────────────────

KPI STRIP:
  Win Rate | Avg Confidence | Best Strategy | Worst Day | Sharpe | Profit Factor
  68.4%    | 78.2%          | Momentum      | -$284     | 2.31   | 1.84

─── 3-COLUMN CHART GRID ─────────────────────────────────────────

[PnL by Strategy — horizontal bar]  [Win Rate by Agent — bar]  [Trade Distribution — donut]

─── AI REASONING BREAKDOWN ──────────────────────────────────────

"Which analyst has been most accurate?"

┌────────────────────────────────────────────────────────────────┐
│  Analyst Accuracy (last 30 days)                               │
│                                                                │
│  Technical  ████████████████░░░  84.2%  (best performer)      │
│  Fundamental ██████████████░░░░  72.1%                        │
│  Sentiment  ████████████░░░░░░░  64.8%                        │
│  News       ████████░░░░░░░░░░░  58.3%  (lowest accuracy)     │
└────────────────────────────────────────────────────────────────┘

─── COORDINATION GRAPH ──────────────────────────────────────────

Visual node graph showing:
  • Each agent as a node
  • Shared token positions drawn as edges
  • Node size = capital allocation
  • Node colour = status (teal=active, amber=paused, red=degraded)
  • Hover: agent name, PnL, confidence

─── STRATEGY CONTRIBUTION ───────────────────────────────────────

Stacked area chart — how much PnL each strategy contributed per week.

─── FLEET HEALTH ────────────────────────────────────────────────

  ████████████████████░░  91%  Fleet Health Score

  Detail breakdown:
  Active agents      :  9 / 12  (75%)
  Avg confidence     :  78.2%
  Degraded agents    :  1 (SpeedBot — latency >2s)
  Agents with losses : 2 (ArbiBot, LITHO/USDC position)
  System latency     :  84ms avg
```

---

### 4.16 Copilot Page

**Route:** `/copilot`  **Auth:** Any

```
FULL-HEIGHT CHAT INTERFACE

─── LEFT PANEL (280px) — Conversation History ───────────────────

  New Conversation  [+]

  Today
  • "Why did AlphaBot buy ETH?"
  • "How is my portfolio doing?"
  • "Stop ArbiBot"

  Yesterday
  • "Best performing agent?"
  • "Explain momentum strategy"

─── RIGHT PANEL — Chat ──────────────────────────────────────────

HEADER:
  🧠 QUANTT Copilot
  "Your AI trading assistant. Ask anything about your portfolio."

CHAT BUBBLES:

  USER:
  ┌────────────────────────────────────────────────────────────┐
  │  Why did AlphaBot buy ETH at 09:41 today?                  │
  └────────────────────────────────────────────────────────────┘

  ASSISTANT (with typing animation then reveal):
  ┌────────────────────────────────────────────────────────────┐
  │  🧠 AlphaBot executed a BUY for ETH/USDC ($1,800) at      │
  │  09:41 today for the following reasons:                    │
  │                                                            │
  │  📉 Technical Analysis (strongest signal, 81/100)         │
  │  RSI was 62 and MACD showed a bullish crossover,           │
  │  suggesting upward momentum.                               │
  │                                                            │
  │  📊 Fundamental (78/100)                                   │
  │  ETH network activity was up 14% MoM with strong           │
  │  developer activity.                                       │
  │                                                            │
  │  The Research Lead found 3 of 4 analysts bullish.          │
  │  Risk Manager confirmed all limits were within bounds.     │
  │  The trade was executed on Uniswap V3 (Arbitrum) at        │
  │  the best available price of $2,940.                       │
  │                                                            │
  │  [View full AI reasoning →]                                │
  └────────────────────────────────────────────────────────────┘

  USER:
  ┌────────────────────────────────────────────────────────────┐
  │  Pause ArbiBot                                             │
  └────────────────────────────────────────────────────────────┘

  ASSISTANT:
  ┌────────────────────────────────────────────────────────────┐
  │  I'll pause ArbiBot for you.                               │
  │                                                            │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │  ⏸  Pause ArbiBot?                                  │  │
  │  │  Strategy: Arbitrage | Chain: Arbitrum               │  │
  │  │  Current position: ETH/USDC $1,200                   │  │
  │  │  [Confirm Pause] [Cancel]                            │  │
  │  └──────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────┘

INPUT BAR:
┌──────────────────────────────────────────────────────────────────┐
│  Ask anything... "Why did X trade?" "Stop all agents" "My PnL"  │
│                                                         [Send ↵] │
└──────────────────────────────────────────────────────────────────┘

SUGGESTED PROMPTS (shown when chat is empty):
  [Why did AlphaBot trade?]  [Show my portfolio summary]
  [Which agent is best?]     [What's the market doing?]
  [Pause all agents]         [Set max loss to $1,000]
```

---

### 4.17 Marketplace Page

**Route:** `/marketplace`  **Auth:** Any

```
PAGE HEADER:
  "Strategy Marketplace"
  "Browse, subscribe, and deploy proven AI trading strategies."

─── SEARCH + FILTERS ────────────────────────────────────────────

[🔍 Search strategies...]  [Category ▼]  [Chain ▼]  [Sort: Top Rated ▼]

─── FEATURED STRATEGIES (hero carousel) ─────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│  ⭐ FEATURED                                                     │
│  🏆 MomentumKing Pro                                            │
│  "Industry-leading momentum strategy. 2.4 Sharpe, 71% win rate" │
│  By @AlexTrades  │  847 subscribers  │  +32.4% last 30d         │
│  [Deploy Strategy →]  [View Details]                            │
└──────────────────────────────────────────────────────────────────┘

─── STRATEGY GRID (3 columns) ───────────────────────────────────

┌──────────────────────────────────────┐
│  📈 MomentumKing Pro          ⭐ 4.9  │
│  by @AlexTrades  ✓ Verified          │
│  ─────────────────────────────────  │
│  Strategy : Momentum                 │
│  Chain    : Arbitrum + Base          │
│  30d PnL  : +32.4%                  │
│  Win Rate : 71%                      │
│  Sharpe   : 2.40                     │
│  Drawdown : -8.2%                    │
│  Subscribers: 847                    │
│  Price    : 2.4 QUANTT/month         │
│  [Deploy →]  [Preview]               │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│  🛡 SafeHedge v2              ⭐ 4.7  │
│  by @RiskFirst                       │
│  ─────────────────────────────────  │
│  Strategy : Hedging                  │
│  30d PnL  : +12.1%                  │
│  Drawdown : -2.1%  (very low)        │
│  Subscribers: 1,241                  │
│  Price    : 1.2 QUANTT/month         │
│  [Deploy →]  [Preview]               │
└──────────────────────────────────────┘

─── LEADERBOARD ─────────────────────────────────────────────────

"Top Traders This Month"

#  | Trader          | Strategy        | 30d PnL | Win Rate | Subscribers
1  | @AlexTrades     | MomentumKing    | +38.4%  | 74%      | 1,284
2  | @DeltaNeutral   | Hedged Arb      | +24.2%  | 68%      | 892
3  | @LithoWhale     | LITHO Momentum  | +21.8%  | 71%      | 441
4  | @SafetraderPro  | Low-risk MR     | +18.4%  | 77%      | 2,841
5  | @RiskFirst      | SafeHedge v2    | +12.1%  | 81%      | 1,241
```

---

### 4.18 Wallet Page

**Route:** `/wallet`  **Auth:** Any authenticated

```
PAGE HEADER:
  "Wallet"
  "Manage your QUANTT token balance and on-chain payments."

─── BALANCE CARD ────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  QUANTT Token Balance                                            │
│                                                                   │
│   ◈ 2,847.42 QUANTT                                             │
│   ≈ $2,847.42 USD                                               │
│                                                                   │
│   Connected: 0x4f3a...8b2c  [Copy] [View on Explorer ↗]         │
│   Chain: Lithosphere (1890)                                      │
│                                                                   │
│   [Send QUANTT]  [Receive]  [Buy QUANTT ↗]                      │
└──────────────────────────────────────────────────────────────────┘

─── TRANSACTION HISTORY ─────────────────────────────────────────

Filters: [All ▼]  [Date range]  [Type: All/Payment/Reward/Transfer]

  Date         | Type      | Description                | Amount       | TX
  2026-05-19   | Payment   | Monthly subscription       | -24.00 QUANTT| 0x9b2c
  2026-05-15   | Reward    | Strategy royalty payout    | +8.40 QUANTT | 0xf41a
  2026-05-01   | Payment   | Invoice #INV-2026-05       | -124.00 QUANTT| 0xa82b
  2026-04-19   | Top-up    | Wallet top-up              | +500.00 QUANTT| 0xc91d
  2026-04-01   | Payment   | Invoice #INV-2026-04       | -98.00 QUANTT | 0xd72e

─── CONNECTED WALLETS ───────────────────────────────────────────

  ┌──────────────────────────────────────────────────────────────┐
  │  0x4f3a...8b2c  │  Lithosphere  │  Primary  [✓]  [Disconnect]│
  └──────────────────────────────────────────────────────────────┘
  [+ Connect Another Wallet]
```

---

### 4.19 Billing Page

**Route:** `/billing`  **Auth:** Any authenticated (ADMIN sees all users)

```
PAGE HEADER:
  "Billing"
  "Manage your subscription, usage, and invoices."

─── CURRENT PLAN ────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│  Current Plan: Pro Trader                    [Upgrade Plan ↗]   │
│  ─────────────────────────────────────────────────────────────  │
│  • Unlimited agents                                              │
│  • Multi-chain trading                                           │
│  • Full insights & analytics                                     │
│  • Priority support                                              │
│  Next billing: Jun 1, 2026   Cost: 48 QUANTT/month             │
└──────────────────────────────────────────────────────────────────┘

─── USAGE THIS MONTH ────────────────────────────────────────────

┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│  Agent Runs         │  │  API Calls           │  │  Est. Cost          │
│  842 / unlimited    │  │  12,420 / unlimited  │  │  82.4 QUANTT        │
│  ████████████░░░░   │  │  ████████░░░░░░░░   │  │  ≈ $82.40           │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘

─── PLAN COMPARISON ─────────────────────────────────────────────

┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
│  Retail Trader     │  │  Pro Operator  ✓   │  │  Enterprise        │
│  FREE              │  │  48 QUANTT/mo      │  │  Custom pricing    │
│  ──────────────   │  │  ──────────────   │  │  ──────────────   │
│  Max 2 agents      │  │  Unlimited agents  │  │  Unlimited agents  │
│  1 chain           │  │  All chains        │  │  All chains        │
│  Basic insights    │  │  Full insights     │  │  Full insights     │
│  Email support     │  │  Priority support  │  │  Dedicated manager │
│                    │  │                    │  │  API access        │
│  [Current Plan]    │  │  [Current ✓]       │  │  [Contact Sales]   │
└────────────────────┘  └────────────────────┘  └────────────────────┘

─── INVOICE HISTORY ─────────────────────────────────────────────

  Invoice         | Period        | Amount       | Status   | Action
  INV-2026-05     | May 2026      | 82.4 QUANTT  | ⏳ Pending| [Pay Now]
  INV-2026-04     | Apr 2026      | 98.0 QUANTT  | ✅ Paid   | [Download]
  INV-2026-03     | Mar 2026      | 91.5 QUANTT  | ✅ Paid   | [Download]
  INV-2026-02     | Feb 2026      | 78.0 QUANTT  | ✅ Paid   | [Download]
  INV-2026-01     | Jan 2026      | 84.2 QUANTT  | ✅ Paid   | [Download]

"Pay Now" → opens modal:
  Amount: 82.4 QUANTT
  From wallet: 0x4f3a...8b2c (Balance: 2,847 QUANTT)
  [Confirm On-Chain Payment]
```

---

### 4.20 Enterprise Fleet Page

**Route:** `/enterprise`  **Auth:** ENTERPRISE + ADMIN

```
PAGE HEADER:
  "Fleet Management"
  "Organisation-wide view of all agents, health, and operations."

─── ORG HEALTH BANNER ───────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│  Apex Trading Corp Fleet                                         │
│  ████████████████████░░  91% Healthy                            │
│  47 agents  │  43 active  │  3 paused  │  1 degraded            │
│  [Emergency Stop All Agents — Red button]                        │
└──────────────────────────────────────────────────────────────────┘

─── FOUR STAT CARDS ─────────────────────────────────────────────

  Total Org Equity | Org PnL 30d | Open Incidents | Avg Latency
  $1,241,800       | +$84,200    | 1 degraded     | 84ms

─── FLEET TABLE ─────────────────────────────────────────────────

Grouped by member:

  ▼ Alex Chen (Owner) — 12 agents
     AlphaBot   | Momentum | Arbitrum | Active  | +$842   | 84%
     EthShield  | Mean Rev | Base     | Active  | +$229   | 71%
     ...

  ▼ Sarah Kim (Member) — 8 agents
     TrendBot   | Momentum | BNB      | Active  | +$1,240 | 88%
     ...

  ▼ James Park (Member) — 27 agents
     ...

─── NODE STATUS PANEL ───────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│  Infrastructure Nodes                                            │
│  ─────────────────────────────────────────────────────────────  │
│  API Gateway     ● Online   99.9%  Latency: 12ms               │
│  AI Engine       ● Online   99.8%  Latency: 284ms              │
│  Arbitrum RPC    ● Online   100%   Latency: 42ms               │
│  Base RPC        ● Online   99.9%  Latency: 38ms               │
│  BNB Chain RPC   ● Online   100%   Latency: 55ms               │
│  Lithosphere RPC ⚠ Degraded 99.1%  Latency: 842ms (elevated)   │
│  Redis           ● Online   100%   Latency: 1ms                │
│  PostgreSQL      ● Online   100%   Latency: 4ms                │
└──────────────────────────────────────────────────────────────────┘

─── API USAGE CHART ─────────────────────────────────────────────

  Bar chart — API calls per day (last 30 days), coloured by API key.
  Total this month: 84,210 calls  |  Rate: 58.5 calls/min peak
```

---

### 4.21 Admin — User Management

**Route:** `/admin/users`  **Auth:** ADMIN only

```
PAGE HEADER:
  "User Management"
  "Create, manage, and assign roles to all platform users."

─── ACTIONS + SEARCH ────────────────────────────────────────────

[+ Create User]  [🔍 Search users...]  [Role ▼]  [Status ▼]  [Export CSV]

─── USER TABLE ──────────────────────────────────────────────────

  #  | Name           | Email                | Role          | Status  | Agents | Joined     | Actions
  1  | Alex Chen      | alex@apex.io         | ENTERPRISE    | Active  | 12     | 2025-10-14 | [Edit] [Role ▼] [Suspend] [Delete]
  2  | Sarah Kim      | sarah@apex.io        | ENT_MEMBER    | Active  | 8      | 2025-11-02 | ...
  3  | James Park     | james@quantt.io      | PRO_OPERATOR  | Active  | 27     | 2025-09-18 | ...
  4  | Mike Johnson   | mike@example.com     | RETAIL_TRADER | Active  | 2      | 2026-01-05 | ...
  5  | Emily Davis    | emily@example.com    | RETAIL_TRADER | Suspended| 0     | 2026-02-14 | ...

─── CREATE USER MODAL ───────────────────────────────────────────

  Full Name       : [text]
  Email           : [email]
  Role            : [Admin / Pro Operator / Retail Trader / Enterprise ▼]
  Organisation    : [None / Apex Trading Corp / ... ▼]  (shown if Enterprise role)
  Send invite email: [✓]

  [Create User]

─── ROLE CHANGE MODAL ───────────────────────────────────────────

  Changing role for: Alex Chen
  Current: PRO_OPERATOR
  New Role: [Admin ▼]
  Reason (optional): [textarea]
  [Confirm Role Change]
```

---

### 4.22 Admin — Organisation Management

**Route:** `/admin/organisations`  **Auth:** ADMIN only

```
─── ORG TABLE ───────────────────────────────────────────────────

  Organisation       | Plan        | Members | Agents | Monthly Spend | Status  | Actions
  Apex Trading Corp  | Enterprise  | 12      | 47     | 1,240 QUANTT  | Active  | [View] [Edit] [Suspend]
  Delta Capital      | Enterprise  | 5       | 18     | 480 QUANTT    | Active  | ...
  RetailFirm LLC     | Pro         | 1       | 4      | 48 QUANTT     | Active  | ...

─── ORG DETAIL PANEL (slide-in) ────────────────────────────────

  Organisation: Apex Trading Corp
  Owner: Alex Chen (alex@apex.io)
  Plan: Enterprise Pro
  API Keys: 3 active
  Sub-users: 12 (1 Owner, 8 Members, 3 Viewers)
  Total Agents: 47
  Monthly Invoice: 1,240 QUANTT
  Status: Active

  [Suspend Org]  [View All Invoices]  [Edit Plan]
```

---

### 4.23 Admin — API Key Management

**Route:** `/admin/api-keys`  **Auth:** ADMIN (all orgs) + ENTERPRISE (own org)

```
PAGE HEADER:
  "API Keys"
  "Manage API keys for programmatic access to QUANTT."

─── CREATE KEY ──────────────────────────────────────────────────

[+ Generate New API Key]

─── KEY TABLE ───────────────────────────────────────────────────

  Label          | Organisation      | Key (masked)   | Scopes       | Calls/mo | Rate Limit | Created    | Status  | Actions
  Production Key | Apex Trading Corp | qtk_live_4f3a• | read, trade  | 84,210   | 1000/min   | 2026-01-15 | Active  | [Revoke]
  Staging Key    | Apex Trading Corp | qtk_test_9b2c• | read         | 12,480   | 100/min    | 2026-03-01 | Active  | [Revoke]
  Read-Only Key  | Delta Capital     | qtk_live_a91f• | read         | 4,200    | 200/min    | 2026-02-10 | Active  | [Revoke]

─── CREATE KEY MODAL ────────────────────────────────────────────

  Key Label       : [text — "Production Key"]
  Organisation    : [Apex Trading Corp ▼]
  Environment     : [Live] [Test]
  Scopes          : [✓ read] [✓ trade] [✓ billing] [admin]
  Rate Limit      : [1000 req/min ▼]
  IP Whitelist    : [optional — 192.168.1.0/24]

  [Generate Key]

After generation:
  ┌──────────────────────────────────────────────────────────────┐
  │  ⚠️  Copy this key now — it will not be shown again.         │
  │                                                              │
  │  qtk_live_4f3a8b2c9d1e2f3a4b5c6d7e8f9a0b1c2d3e4f         │
  │  [Copy Key]                                                  │
  └──────────────────────────────────────────────────────────────┘
```

---

### 4.24 Settings Page

**Route:** `/settings`  **Auth:** Any authenticated

```
─── SIDEBAR TABS ────────────────────────────────────────────────

  [Profile]  [Security]  [Notifications]  [Risk Defaults]  [Connected Wallets]

─── TAB: PROFILE ────────────────────────────────────────────────

  Avatar upload (drag & drop circle)
  Full Name:     [Alex Chen]
  Email:         [alex@quantt.io]  [Verified ✓]
  Role:          Pro Operator  (read-only, changed by admin)
  Member since:  September 2025
  [Save Changes]

─── TAB: SECURITY ───────────────────────────────────────────────

  Password
  Current: [••••••••••]
  New:     [••••••••••]
  Confirm: [••••••••••]
  [Update Password]

  Two-Factor Authentication (2FA)
  Status: Not enabled
  [Enable 2FA with Authenticator App]
  → QR code + backup codes shown after enable

  Active Sessions
  │  Browser    │  IP            │  Location       │ Last Active │ Action    │
  │  Chrome 124 │  192.168.1.100 │  Mumbai, IN     │ Now         │ [Current] │
  │  Safari 17  │  103.41.2.180  │  Delhi, IN      │ 2h ago      │ [Revoke]  │

  [Sign Out All Other Sessions]

─── TAB: NOTIFICATIONS ──────────────────────────────────────────

  Notification Preferences

  Category          │ In-App │ Email │ Push
  Trade executed    │  ✓     │  ✓   │  ✓
  Risk limit hit    │  ✓     │  ✓   │  ✓
  Agent degraded    │  ✓     │  ✓   │  ✓
  Daily PnL report  │  ✓     │  ✓   │  ○
  Invoice due       │  ✓     │  ✓   │  ✓
  Market alerts     │  ✓     │  ○   │  ○
  System updates    │  ✓     │  ○   │  ○

  [Save Preferences]

─── TAB: RISK DEFAULTS ──────────────────────────────────────────

  Default settings applied to all newly created agents:

  Default Risk Level     : [Moderate ▼]
  Default Max Daily Loss : [$500]
  Default Max Position   : [$2,500]
  Default Max Drawdown   : [15%]
  Default Autopilot      : [OFF]

  Global Kill Switch     : [OFF ○────] ← turns red, pauses ALL agents

  [Save Defaults]

─── TAB: CONNECTED WALLETS ──────────────────────────────────────

  ┌──────────────────────────────────────────────────────────────┐
  │  0x4f3a...8b2c  │  Lithosphere  │  Primary ✓  │ [Disconnect] │
  └──────────────────────────────────────────────────────────────┘
  [+ Connect Wallet]
```

---

### 4.25 Alerts Page

**Route:** `/alerts`  **Auth:** Any authenticated

```
PAGE HEADER:
  "Alerts"  [Mark All Read]  [Settings ⚙]

─── FILTER TABS ─────────────────────────────────────────────────

[All (12)] [Trade (4)] [Risk (3)] [System (2)] [Billing (1)] [Market (2)]

─── ALERT LIST ──────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────────┐
│  🔴 RISK  •  CRITICAL  •  2 mins ago                        [Ack ✓] │
│  AlphaBot: Daily loss limit 82% reached ($411 of $500)               │
│  Consider pausing AlphaBot or reducing position size.                │
│  [View Agent →]                                                       │
├──────────────────────────────────────────────────────────────────────┤
│  🟢 TRADE  •  INFO  •  5 mins ago                           [Ack ✓] │
│  AlphaBot executed BUY ETH/USDC $1,800 at $2,940  +2.1% expected    │
│  [View Trade →]                                                       │
├──────────────────────────────────────────────────────────────────────┤
│  🟡 SYSTEM  •  WARNING  •  22 mins ago                      Acked ✓  │
│  SpeedBot response latency elevated (842ms vs 200ms avg)             │
│  Agent performance may be degraded.                                   │
├──────────────────────────────────────────────────────────────────────┤
│  🔵 BILLING  •  INFO  •  2 days ago                         Acked ✓  │
│  Invoice INV-2026-05 is due June 1, 2026.  Amount: 82.4 QUANTT       │
│  [Pay Now →]                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  🟡 MARKET  •  INFO  •  3 days ago                          Acked ✓  │
│  ETH/USDC: 5.2% price movement detected — agents re-evaluating       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Reusable Component Library

### 5.1 MetricCard

```
Props: label, value, delta?, deltaLabel?, trend?: 'up'|'down'|'neutral', icon?, accentColour?

Visual:
┌────────────────────────────┐
│  [icon]  label             │
│                            │
│  $12,847.32                │  ← --text-h2, --text-primary
│  ↑ +3.21%  vs yesterday   │  ← success/danger colour
└────────────────────────────┘
Hover: subtle teal border glow
```

### 5.2 AgentCard

```
Props: agent (AgentSummary), onRun, onPause, onView, readonly?

Status badge top-right with colour and pulsing dot for Active.
PnL shown in success/danger colour.
Action buttons hidden if readonly.
```

### 5.3 StatusBadge

```
Props: status: 'active'|'paused'|'learning'|'degraded'|'idle'|'error'

Colours:
  active   → bg --success-dim, text --success, pulsing dot
  paused   → bg --warning-dim, text --warning
  learning → bg --purple-dim, text --purple
  degraded → bg --danger-dim, text --danger, pulsing dot
  idle     → bg rgba(255,255,255,0.05), text --text-muted
  error    → bg --danger-dim, text --danger, urgent pulsing
```

### 5.4 ChartCard

```
Props: title, children (chart), timeRangeTabs?, actions?

Wrapper providing:
  - Consistent padding and border
  - Title + optional time range selector tabs
  - Loading skeleton state
  - Empty state illustration
```

### 5.5 AnalystCard

```
Props: analyst: 'fundamental'|'sentiment'|'news'|'technical', score, verdict, reasoning

Collapsible card with coloured left border:
  fundamental → amber
  sentiment   → purple
  news        → blue
  technical   → teal
Score shown as circular progress ring + number.
```

### 5.6 AlertItem

```
Props: alert (AlertItem), onAck, onClick

Left coloured border by severity:
  critical → danger red
  warning  → amber
  info     → blue
  success  → teal

Unacknowledged: slightly brighter background
Acknowledged: dimmer, no action button
```

### 5.7 DataTable

```
Props: columns[], rows[], loading?, onSort?, pagination?

Features:
  - Sticky header
  - Sortable columns (click header to sort)
  - Row hover state (--bg-hover)
  - Loading skeleton (3 rows of shimmering placeholders)
  - Empty state with icon and message
  - Pagination controls
  - Row click to open detail
```

### 5.8 CommandPalette (⌘K)

```
Global overlay triggered by ⌘K / Ctrl+K

┌──────────────────────────────────────────────────────────────┐
│  🔍  Search agents, pages, tokens...                         │
├──────────────────────────────────────────────────────────────┤
│  AGENTS                                                      │
│  🤖  AlphaBot     →  Active  │  +$842    [View]             │
│  🤖  EthShield    →  Active  │  +$229    [View]             │
│                                                              │
│  PAGES                                                       │
│  📊  Dashboard    →  /dashboard                             │
│  📈  Market Data  →  /market                                │
│  💡  Insights     →  /insights                              │
└──────────────────────────────────────────────────────────────┘
```

### 5.9 KillSwitchModal

```
Confirmation modal triggered by Kill Switch button.

┌──────────────────────────────────────────────────────────────┐
│  ⚠️  Emergency Stop All Agents                               │
│                                                              │
│  This will immediately pause ALL 12 active agents.           │
│  Open positions will NOT be closed automatically.            │
│  You can restart agents individually after.                  │
│                                                              │
│  Type "STOP ALL" to confirm:                                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  STOP ALL                                              │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  [Cancel]           [🔴 Stop All Agents]                    │
└──────────────────────────────────────────────────────────────┘
```

### 5.10 RoleBadge (POC Dev Tool)

```
Shown top-right in POC mode:
  "👤 Viewing as: [Admin ▼]  POC"

Clicking the dropdown switches role instantly.
Persisted in localStorage for the session.
```

---

## 6. Static Data Payloads

### 6.1 Users (one per role)

```json
[
  {
    "id": "usr_admin_001",
    "name": "Diana Reeves",
    "email": "admin@quantt.io",
    "role": "ADMIN",
    "avatarUrl": "https://i.pravatar.cc/80?u=admin",
    "joinedAt": "2025-08-01T00:00:00Z"
  },
  {
    "id": "usr_pro_001",
    "name": "James Park",
    "email": "pro@quantt.io",
    "role": "PRO_OPERATOR",
    "avatarUrl": "https://i.pravatar.cc/80?u=pro",
    "joinedAt": "2025-09-18T00:00:00Z"
  },
  {
    "id": "usr_retail_001",
    "name": "Alex Chen",
    "email": "retail@quantt.io",
    "role": "RETAIL_TRADER",
    "avatarUrl": "https://i.pravatar.cc/80?u=retail",
    "joinedAt": "2026-01-05T00:00:00Z"
  },
  {
    "id": "usr_ent_001",
    "name": "Sarah Kim",
    "email": "enterprise@quantt.io",
    "role": "ENTERPRISE",
    "organisation": "Apex Trading Corp",
    "avatarUrl": "https://i.pravatar.cc/80?u=ent",
    "joinedAt": "2025-10-14T00:00:00Z"
  }
]
```

### 6.2 Portfolio Summary (by role)

```json
{
  "RETAIL_TRADER": {
    "totalEquityUsd": 12847.32,
    "pnl24hUsd": 411.20,
    "pnl24hPct": 3.21,
    "pnl7dUsd": 1241.00,
    "pnl30dUsd": 2847.32,
    "sharpe30d": 1.84,
    "maxDrawdown30d": -4.2,
    "activeAgents": 2,
    "agentLimit": 2
  },
  "PRO_OPERATOR": {
    "totalEquityUsd": 284720.00,
    "pnl24hUsd": 9241.00,
    "pnl24hPct": 3.36,
    "pnl30dUsd": 42800.00,
    "sharpe30d": 2.31,
    "maxDrawdown30d": -6.8,
    "activeAgents": 9,
    "agentLimit": -1,
    "winRate": 67.4,
    "modelConfidence": 84
  },
  "ENTERPRISE": {
    "totalEquityUsd": 1241800.00,
    "pnl24hUsd": 41200.00,
    "pnl30dUsd": 128400.00,
    "sharpe30d": 2.84,
    "maxDrawdown30d": -5.2,
    "activeAgents": 43,
    "totalAgents": 47,
    "apiCallsThisMonth": 84210,
    "monthlySpendQUANTT": 1240
  },
  "ADMIN": {
    "platformTotalUsers": 3241,
    "platformActiveAgents": 8847,
    "platformEquityUsd": 284000000,
    "platform24hVolumeUsd": 42000000,
    "activeOrganisations": 127,
    "apiKeys": 342,
    "pendingInvoices": 18,
    "systemHealth": 100
  }
}
```

### 6.3 Agents (full list — 12 agents)

```json
[
  {
    "id": "agt_001",
    "name": "AlphaBot",
    "strategy": "momentum",
    "chains": ["arbitrum"],
    "tokens": ["ETH", "WBTC"],
    "capitalUsd": 10000,
    "status": "active",
    "riskLevel": "moderate",
    "autopilot": true,
    "pnl30dUsd": 842.10,
    "pnl30dPct": 8.42,
    "confidence": 84,
    "totalTrades": 142,
    "winRate": 68.4,
    "lastRunAt": "2026-05-19T09:41:27Z",
    "maxDailyLossUsd": 500,
    "maxPositionUsd": 2500,
    "maxDrawdownPct": 15,
    "uptimePct": 99.2
  },
  {
    "id": "agt_002",
    "name": "EthShield",
    "strategy": "mean_reversion",
    "chains": ["base"],
    "tokens": ["ETH", "USDC"],
    "capitalUsd": 5000,
    "status": "active",
    "riskLevel": "low",
    "autopilot": false,
    "pnl30dUsd": 229.80,
    "pnl30dPct": 4.59,
    "confidence": 71,
    "totalTrades": 84,
    "winRate": 72.6,
    "lastRunAt": "2026-05-19T09:30:00Z"
  },
  {
    "id": "agt_003",
    "name": "ArbiBot",
    "strategy": "arbitrage",
    "chains": ["arbitrum"],
    "tokens": ["ETH", "WBTC", "USDC"],
    "capitalUsd": 8000,
    "status": "paused",
    "riskLevel": "high",
    "pnl30dUsd": -84.20,
    "pnl30dPct": -1.05,
    "confidence": null,
    "totalTrades": 210,
    "winRate": 61.4
  },
  {
    "id": "agt_004",
    "name": "GhostArb",
    "strategy": "arbitrage",
    "chains": ["base"],
    "tokens": ["ETH", "USDC", "DAI"],
    "capitalUsd": 12000,
    "status": "active",
    "pnl30dUsd": 1284.00,
    "pnl30dPct": 10.7,
    "confidence": 91
  },
  {
    "id": "agt_005",
    "name": "LithoTrader",
    "strategy": "momentum",
    "chains": ["lithosphere"],
    "tokens": ["LITHO", "USDC"],
    "capitalUsd": 3000,
    "status": "active",
    "pnl30dUsd": 412.50,
    "pnl30dPct": 13.75,
    "confidence": 78
  },
  {
    "id": "agt_006",
    "name": "BNBMatic",
    "strategy": "trend_following",
    "chains": ["bnb"],
    "tokens": ["BNB", "USDT"],
    "capitalUsd": 6000,
    "status": "learning",
    "pnl30dUsd": 0,
    "pnl30dPct": 0,
    "confidence": 62
  },
  {
    "id": "agt_007",
    "name": "SafeHold",
    "strategy": "hedging",
    "chains": ["arbitrum"],
    "tokens": ["ETH", "WBTC", "USDC"],
    "capitalUsd": 20000,
    "status": "active",
    "pnl30dUsd": 320.00,
    "pnl30dPct": 1.6,
    "confidence": 68
  },
  {
    "id": "agt_008",
    "name": "SpeedBot",
    "strategy": "momentum",
    "chains": ["bnb"],
    "tokens": ["BNB", "CAKE"],
    "capitalUsd": 4000,
    "status": "degraded",
    "pnl30dUsd": 142.00,
    "pnl30dPct": 3.55,
    "confidence": 55
  },
  {
    "id": "agt_009",
    "name": "ValueSeeker",
    "strategy": "fundamental",
    "chains": ["base"],
    "tokens": ["ETH", "LINK", "AAVE"],
    "capitalUsd": 7500,
    "status": "active",
    "pnl30dUsd": 680.00,
    "pnl30dPct": 9.07,
    "confidence": 76
  },
  {
    "id": "agt_010",
    "name": "DeltaNeutral",
    "strategy": "hedging",
    "chains": ["arbitrum"],
    "tokens": ["ETH", "WBTC"],
    "capitalUsd": 15000,
    "status": "paused",
    "pnl30dUsd": 210.00,
    "pnl30dPct": 1.4,
    "confidence": null
  },
  {
    "id": "agt_011",
    "name": "MACDRunner",
    "strategy": "technical",
    "chains": ["base"],
    "tokens": ["ETH", "USDC"],
    "capitalUsd": 5000,
    "status": "active",
    "pnl30dUsd": 390.00,
    "pnl30dPct": 7.8,
    "confidence": 81
  },
  {
    "id": "agt_012",
    "name": "RiskOff",
    "strategy": "mean_reversion",
    "chains": ["lithosphere"],
    "tokens": ["LITHO", "USDC"],
    "capitalUsd": 2500,
    "status": "idle",
    "pnl30dUsd": 0,
    "pnl30dPct": 0,
    "confidence": null
  }
]
```

### 6.4 Activity Feed (20 events)

```json
[
  {"id":"evt_001","agentId":"agt_001","agentName":"AlphaBot","type":"trade","severity":"info","title":"BUY ETH/USDC executed","detail":"$1,800 @ $2,940 — Uniswap V3 — Slippage 0.08%","txHash":"0x4f3a8b2c9d1e2f3a4b5c6d7e8f9a0b1c","timestamp":"2026-05-19T09:42:10Z"},
  {"id":"evt_002","agentId":"agt_001","agentName":"AlphaBot","type":"risk","severity":"info","title":"Risk check PASSED","detail":"All 4 limits within bounds","timestamp":"2026-05-19T09:41:55Z"},
  {"id":"evt_003","agentId":"agt_001","agentName":"AlphaBot","type":"decision","severity":"info","title":"Trader: BUY decision","detail":"Buy ETH $1,800. Confidence 84%","timestamp":"2026-05-19T09:41:50Z"},
  {"id":"evt_004","agentId":"agt_001","agentName":"AlphaBot","type":"research","severity":"info","title":"Research Lead: BULLISH consensus","detail":"3/4 analysts agree. Technical signal strongest.","timestamp":"2026-05-19T09:41:45Z"},
  {"id":"evt_005","agentId":"agt_001","agentName":"AlphaBot","type":"analysis","severity":"info","title":"4 analysts completed (parallel)","detail":"Fundamental 78 | Sentiment 72 | News 65 | Technical 81","timestamp":"2026-05-19T09:41:30Z"},
  {"id":"evt_006","agentId":"agt_002","agentName":"EthShield","type":"risk","severity":"warning","title":"Daily loss 82% of limit reached","detail":"$411 of $500 daily limit used","timestamp":"2026-05-19T09:38:00Z"},
  {"id":"evt_007","agentId":"agt_004","agentName":"GhostArb","type":"trade","severity":"info","title":"SELL WBTC/USDC executed","detail":"$500 @ $62,100 — SushiSwap — +$45.00","txHash":"0x9b2cf41a","timestamp":"2026-05-19T07:31:00Z"},
  {"id":"evt_008","agentId":"agt_008","agentName":"SpeedBot","type":"system","severity":"warning","title":"High latency detected","detail":"842ms response (avg 200ms). Performance degraded.","timestamp":"2026-05-19T06:20:00Z"}
]
```

### 6.5 Market Data

```json
{
  "ETH": {
    "symbol": "ETH",
    "name": "Ethereum",
    "priceUsd": 2947.42,
    "change24hPct": 2.41,
    "volume24hUsd": 18400000000,
    "marketCapUsd": 354000000000,
    "rsi": 62.4,
    "macdSignal": "BUY",
    "macdHistogram": 12.4,
    "fearGreedIndex": 72,
    "sentimentScore": 72,
    "fundamentalScore": 78
  },
  "BTC": {
    "symbol": "BTC",
    "priceUsd": 67240,
    "change24hPct": 1.82,
    "rsi": 58.1,
    "macdSignal": "NEUTRAL",
    "fearGreedIndex": 68
  },
  "LITHO": {
    "symbol": "LITHO",
    "priceUsd": 0.82,
    "change24hPct": 5.1,
    "rsi": 71.2,
    "macdSignal": "BUY"
  },
  "BNB": {
    "symbol": "BNB",
    "priceUsd": 612.30,
    "change24hPct": -0.41,
    "rsi": 48.3,
    "macdSignal": "NEUTRAL"
  }
}
```

### 6.6 Invoices

```json
[
  {"id":"INV-2026-05","period":"May 2026","amountQUANTT":82.4,"status":"pending","dueDate":"2026-06-01"},
  {"id":"INV-2026-04","period":"Apr 2026","amountQUANTT":98.0,"status":"paid","paidAt":"2026-04-30","txHash":"0xa82b"},
  {"id":"INV-2026-03","period":"Mar 2026","amountQUANTT":91.5,"status":"paid","paidAt":"2026-03-31","txHash":"0xc91d"},
  {"id":"INV-2026-02","period":"Feb 2026","amountQUANTT":78.0,"status":"paid","paidAt":"2026-02-28","txHash":"0xd72e"},
  {"id":"INV-2026-01","period":"Jan 2026","amountQUANTT":84.2,"status":"paid","paidAt":"2026-01-31","txHash":"0xe83f"}
]
```

### 6.7 Copilot Responses (static reply map)

```json
{
  "portfolio": "Your portfolio is valued at $12,847.32 today, up +3.21% from yesterday. AlphaBot has been your strongest performer with +$842 in 30 days.",
  "alphabot_trade": "AlphaBot bought ETH/USDC ($1,800) at 09:41 based on a Technical Analyst BUY signal (RSI 62, MACD bullish crossover) supported by strong Fundamental and Sentiment scores. Risk Manager approved the trade.",
  "best_agent": "Your best performing agent is GhostArb with +$1,284 (10.7%) in 30 days, trading on Base with 91% model confidence.",
  "pause_agent": "I can pause that agent for you. Please confirm the action below.",
  "stop_all": "I'll initiate an emergency stop on all agents. Please type STOP ALL to confirm.",
  "market": "ETH is trading at $2,947 (+2.41%). RSI is 62.4 (not overbought), MACD shows a bullish crossover. Fear & Greed Index is 72 (Greed). Sentiment is positive."
}
```

---

## 7. Animation & Interaction Spec

### 7.1 Page Transitions
```
Route change: fade-in (200ms ease-out) + slight upward slide (8px → 0px)
```

### 7.2 Card Entrance
```
Staggered entrance on page load:
  Each card delays by index × 60ms
  Animation: opacity 0→1 + translateY(12px→0) in 300ms ease-out
```

### 7.3 Number Counter Animation
```
KPI metric values count up from 0 to final value over 800ms on page load.
Use ease-out curve. Only fires once per session.
```

### 7.4 Live Feed Events
```
New event slides in from top: translateY(-20px)→0 + opacity 0→1 in 250ms.
After 30 events, oldest slides out from bottom.
```

### 7.5 Status Dot Pulse
```
Active / Degraded status dots:
  box-shadow animation: 0→8px spread in --success / --danger colour, 1.5s infinite
  keyframes: 0%{opacity:1} 50%{opacity:0.4} 100%{opacity:1}
```

### 7.6 Chart Animations
```
Area/line charts: draw animation left-to-right over 600ms on mount.
Bar charts: grow from height:0 staggered by 40ms per bar.
Donut charts: arc draws clockwise from 0 over 500ms.
```

### 7.7 Skeleton Loading
```
All data tables and cards show skeleton placeholder while loading.
Skeleton: background linear-gradient shimmer animation (2s infinite).
Shimmer colour: from --bg-surface to --bg-hover to --bg-surface.
```

### 7.8 Toast Notifications
```
Position: top-right, 16px margin.
Enter: slide in from right (240px→0) + fade in, 200ms.
Exit: slide out to right + fade out, 150ms.
Duration: success=3s, warning=5s, error=stays until dismissed.
```

### 7.9 Hover States
```
Sidebar nav items   : bg transitions to --bg-hover over 150ms
Table rows          : bg transitions to --bg-hover over 100ms
Cards               : border-color transitions to --border-active over 150ms
Buttons             : transform: scale(0.98) + bg colour shift, 100ms
Action icon buttons : colour transitions to --primary over 150ms
```

### 7.10 Kill Switch Button
```
Normal state: grey ghost button, subtle
Hover state: red border glow appears (transition 200ms), text turns danger red
Clicked: modal opens with confirmation input
```

---

## 8. Responsive Breakpoints

```
xs  : < 480px   — mobile (not the target for web POC, but don't break)
sm  : 480-768px — tablet
md  : 768-1024px — laptop
lg  : 1024-1280px — desktop (primary target)
xl  : 1280-1536px — wide desktop
2xl : > 1536px  — ultrawide
```

### Sidebar behaviour
```
< 1024px : sidebar collapses to icon-only (64px) by default
≥ 1024px : sidebar expanded (240px) by default, user can collapse
< 768px  : sidebar becomes bottom sheet / hamburger overlay
```

### Dashboard grid
```
KPI cards: 
  2xl/xl : 4 across (or 6 for PRO/ADMIN)
  lg/md  : 2 across
  sm     : 1 across

Charts:
  lg+   : side by side (60/40 split)
  md-   : stacked full-width

Agent grid:
  xl+   : 3 columns
  lg/md : 2 columns
  sm    : 1 column
```

---

*End of QUANTT POC Design Document*
*Total screens: 25 | Total components: 10 core | Static data payloads: 7 | Roles covered: 6*
