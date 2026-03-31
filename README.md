# 🔮 El Oraculo

<p align="center">
  <img src="assets/banner.jpg" alt="El Oraculo — Autonomous AI Trading Enhancement Engine" width="100%">
</p>

**Autonomous AI enhancement engine for crypto grid trading bots.**

El Oraculo sits alongside your grid trading bot and makes it smarter. The free tier gives you the signal pipeline, watchdog, conflict resolver, confidence tracker, and basic backtester. **[El Oraculo Pro](https://buy.stripe.com/cNi4gB8ZWfVE5U7cpY5J600)** adds HMM Markov regime detection, Nemotron LLM predictions, Karpathy autoresearch, and self-evolving goals.

Built for [Binance Futures](https://www.binance.com/en/futures) grid/mean-reversion strategies. Works with any bot that exposes a REST API for parameter overrides.

---

## Features

- **🧬 HMM Markov Regime Detection** — 2-state probabilistic model trained on 30-day candle data. Detects "normal" vs "spike" regimes with 96-98% sticky state probability. Reduces false regime transitions by 29-100% compared to threshold-based ADX guards.

- **🤖 LLM Market Predictions** — Nvidia Nemotron 120B (free via OpenRouter) analyzes RSI, MACD, ADX, Hurst exponent, BB squeeze, funding rates, and news sentiment. Returns structured JSON predictions with calibrated confidence scores.

- **🔬 Autoresearch Optimizer** — Adapted from [Karpathy's autoresearch](https://github.com/karpathy/autoresearch). Autonomously proposes parameter changes, backtests them against 7 days of candle data with 30-day pattern analysis, and keeps improvements above 2%. Runs 20 iterations per 12-hour cycle.

- **📊 Faithful Backtest Engine** — Simulates grid trading with ADX guard (4-tier), risk guards (drawdown halt, anti-tilt cooldowns), fee profitability guard, and position abandonment on reposition. Produces realistic 80-87% win rates, not fantasy 100%.

- **📈 Self-Evolving Goals** — Weekly revenue targets sourced from Binance income API. Exceeded target? Compound upward 10%. Missed? Diagnose and recalibrate. The system raises its own bar.

- **🛡️ Safety Guards** — Rate limiting (1 change per param per 5 min), bounds checking (spacing 0.25%-0.60%), max 15% change per application, parameter whitelist, monitor-only watchdog. Cannot change leverage, risk guards, or trading pairs.

- **📡 Signal Pipeline** — Conflict resolution when modules disagree (higher confidence wins, reduced 20%). Confidence compounding (1.0x → 1.5x on win streaks, dampens on misses, pauses after 3 consecutive failures).

- **🎯 Revenue Attribution** — Every signal gets a UUID. Track which module generates money. Skills that produce revenue get promoted; skills that don't get deprecated.

---

## Architecture

```
┌──────────────────────────────────┐         ┌──────────────────────────┐
│  El Oraculo (Node.js)            │         │  Your Trading Bot        │
│                                  │         │                          │
│  Orchestrator                    │ signals │  REST API                │
│  ├── Watchdog (30s, monitor)     │────────>│  POST /api/apply-param   │
│  ├── Signal Processor (60s)      │         │  GET  /api/status        │
│  ├── Conflict Resolver           │         │  GET  /api/indicators    │
│  └── Confidence Tracker          │         │                          │
│                                  │ reads   │  Trading Database        │
│  HMM Regime (1h)                 │<────────│  (SQLite, read-only)     │
│  ├── 2-state GaussianHMM        │         │                          │
│  └── Forward filter              │         └──────────────────────────┘
│                                  │
│  Nemotron LLM (4h)              │         ┌──────────────────────────┐
│  ├── OpenRouter API              │         │  Relay Server            │
│  └── Structured predictions      │────────>│  ├── Binance proxy (RO)  │
│                                  │         │  └── Signal applicator   │
│  Autoresearch (12h)              │         └──────────────────────────┘
│  ├── Pattern analyzer (30-day)   │
│  ├── Backtest engine (faithful)  │         ┌──────────────────────────┐
│  └── Karpathy optimize loop     │         │  Binance Futures API     │
│                                  │<────────│  (source of truth)       │
│  Skill Evolution                 │  income │  /fapi/v1/income         │
│  └── Revenue tracking           │         │  /fapi/v2/balance        │
└──────────────────────────────────┘         └──────────────────────────┘
```

### Signal Flow

```
LLM/HMM/Autoresearch → Signal JSON → Conflict Resolver → Confidence Gate → Relay → Bot API → State Table → Grid Engine
```

**Confidence tiers:**
| Confidence | Action |
|-----------|--------|
| < 0.40 | Log only |
| 0.40 - 0.60 | Apply 50% (conservative) |
| 0.60 - 0.80 | Apply 100% |
| > 0.80 | Apply + widen exploration |

---

## Quick Start

### Prerequisites
- Node.js 20+
- Python 3.10+ with `hmmlearn` (`pip install hmmlearn numpy scipy`)
- A grid trading bot with REST API (e.g., [ccxt](https://github.com/ccxt/ccxt)-based)
- OpenRouter API key (free at [openrouter.ai](https://openrouter.ai))

### Install

```bash
git clone https://github.com/Niiks7777/el-oraculo.git
cd el-oraculo
npm install
cp .env.example .env
# Edit .env with your API keys and bot URL
```

### Configure

Edit `.env`:
```env
EL_PESOS_API=http://localhost:4201    # Your trading bot's API
RELAY_API=http://localhost:4202        # Relay server
OPENROUTER_API_KEY=sk-or-v1-...       # Free from openrouter.ai
TELEGRAM_BOT_TOKEN=...                # Optional: alerts
TELEGRAM_CHAT_ID=...                  # Optional: alerts
```

### Build & Run

```bash
# Build
npm run build

# Train HMM models (needs candle data — fetched automatically on first run)
npm run train:hmm

# Start the main engine
npm start

# Start the relay (on the same machine as your trading bot)
npm run start:relay
```

### Dashboard

Open `http://localhost:4203` for the El Oraculo command center.

---

## Modules

### Optimizer (`src/optimizer/`)

| File | Purpose |
|------|---------|
| `backtest-engine.ts` | Faithful grid simulator with ADX guard, risk guards, fee guard |
| `hmm-filter.ts` | Forward algorithm for HMM state probabilities |
| `hmm-trainer.py` | Trains 2-state GaussianHMM on hourly log-returns |
| `hmm-signal-generator.ts` | Generates spacing signals from HMM regime state |
| `loop-runner.ts` | Karpathy autoresearch loop (propose → backtest → keep/revert) |
| `pattern-analyzer.ts` | 30-day deep analysis (volatility regimes, ToD, mean reversion) |
| `scoring.ts` | Scoring aligned with real trading: 0.6×fillRate + 0.4×pnlPerTrip |
| `param-space.ts` | Parameter bounds and constraints |
| `history-loader.ts` | Fetches and caches Binance 1m candles |
| `compare-regimes.ts` | Backtest comparison: ADX thresholds vs HMM |

### Orchestrator (`src/orchestrator/`)

| File | Purpose |
|------|---------|
| `scheduler.ts` | Runs all modules on schedule (30s/60s/1h/4h/12h/weekly) |
| `signal-bus.ts` | JSON signal file management with TTL |
| `conflict-resolver.ts` | When signals disagree, higher confidence wins (-20%) |
| `confidence-tracker.ts` | Compounds on wins (→1.5x), dampens on losses (→0.5x) |
| `goal-system.ts` | Self-evolving weekly targets from Binance income |
| `watchdog.ts` | Health monitoring (alerts only, never restarts/kills) |
| `telegram-sender.ts` | Notification delivery |
| `dashboard.ts` | Express API + command center UI at :4203 |

### Predictor (`src/predictor/`)

| File | Purpose |
|------|---------|
| `llm-predictor.ts` | Nemotron 120B via OpenRouter — structured market predictions |
| `market-feeder.ts` | Aggregates indicators, funding, sentiment into LLM context |

### Collector (`src/collector/`)

| File | Purpose |
|------|---------|
| `api-reader.ts` | Reads 25+ trading bot REST API endpoints |
| `db-reader.ts` | Read-only SQLite queries on trading database |

### Relay (`src/relay/`)

| File | Purpose |
|------|---------|
| `server.ts` | Binance API proxy (read-only) + signal applicator |

---

## HMM Markov Model

El Oraculo uses a 2-state Hidden Markov Model trained on hourly log-returns:

- **State 0 (Normal):** Low volatility, mean-reverting. Grid runs tight spacing.
- **State 1 (Spike):** High volatility event. Grid widens spacing or pauses.

The model learns transition probabilities from 30 days of candle data:

```
         Normal → Normal:  96.8%    Normal → Spike:  3.2%
         Spike  → Normal:  89.8%    Spike  → Spike: 10.2%
```

The forward filter computes `P(state | observations)` at each hour. Signals are only generated when confidence exceeds 70%, preventing false regime transitions (whipsaw).

**Backtest results (HMM vs ADX thresholds):**

| Metric | ADX | HMM | Delta |
|--------|-----|-----|-------|
| Win rate (SOL) | 77.9% | 87.0% | **+9.1%** |
| Win rate (TAO) | 85.9% | 91.4% | **+5.5%** |
| Abandoned positions (TAO) | 62 | 44 | **-29%** |
| Regime changes (TAO) | 62 | 0 | **-100%** |

---

## Safety

El Oraculo is designed to enhance, never endanger:

| Guard | Description |
|-------|-------------|
| **Parameter whitelist** | Can only change `spacing`, `allocation`, `atr_mult` |
| **Bounds checking** | Spacing: 0.25%-0.60%, Allocation: $15-$80 |
| **Rate limiting** | Max 1 change per parameter per 5 minutes |
| **Max change** | 15% maximum per application |
| **Monitor-only watchdog** | Alerts on failure, never restarts or kills positions |
| **Independent risk guards** | Your bot's own risk system runs independently |
| **Graceful degradation** | If Oraculo dies, your bot continues on its own |

---

## Configuration

All configuration via environment variables (see `.env.example`).

| Variable | Default | Description |
|----------|---------|-------------|
| `EL_PESOS_API` | `http://localhost:4201` | Your trading bot's API URL |
| `RELAY_API` | `http://localhost:4202` | Relay server URL |
| `ORACULO_PORT` | `4203` | Dashboard port |
| `OPENROUTER_API_KEY` | — | Free key from openrouter.ai |
| `OPENROUTER_DAILY_CAP` | `5` | Daily spend cap in USD |
| `OLLAMA_URL` | `http://localhost:11434` | Local LLM (optional) |
| `PGVECTOR_URL` | — | PostgreSQL with pgvector (optional) |
| `TELEGRAM_BOT_TOKEN` | — | Telegram alerts (optional) |
| `TELEGRAM_CHAT_ID` | — | Telegram chat (optional) |

---

## Integrating with Your Bot

El Oraculo needs your trading bot to expose:

1. **`GET /api/status`** — Returns bot state (positions, balance, regime, indicators)
2. **`GET /api/indicators`** — Returns RSI, MACD, ADX, Hurst, etc. per pair
3. **`POST /api/apply-param`** — Accepts `{ parameter, symbol, newValue, source, signalId }` and applies the parameter override

The relay server proxies Binance API calls for P&L tracking (read-only, source of truth).

---

## Inspired By

- [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) — autonomous experiment loop
- [HKUDS OpenSpace](https://github.com/HKUDS/OpenSpace) — self-evolving skill engine
- [CAMEL-AI OASIS](https://github.com/camel-ai/oasis) — multi-agent social simulation

---

## Free vs Pro

| Feature | Free | Pro ($149) |
|---------|------|-----------|
| Signal pipeline + conflict resolver | Yes | Yes |
| Confidence compounding | Yes | Yes |
| Watchdog monitoring | Yes | Yes |
| Telegram alerts | Yes | Yes |
| Basic backtester | Yes | Yes |
| Pattern analyzer | Yes | Yes |
| Dashboard | Basic | Full command center |
| **HMM Markov regime detection** | - | **+9% win rate** |
| **Nemotron LLM predictions** | - | **Free API, real AI inference** |
| **Karpathy autoresearch** | - | **20 iterations/cycle** |
| **Faithful backtest engine** | - | **ADX guard, risk guards** |
| **Self-evolving goals** | - | **Compound weekly targets** |
| **Revenue attribution** | - | **Track which module makes money** |
| **Skill evolution** | - | **Auto-promote/deprecate strategies** |

**[Get El Oraculo Pro — 549 AED](https://buy.stripe.com/cNi4gB8ZWfVE5U7cpY5J600)** — one-time purchase, lifetime updates, 30 days email support.

**[Pro + Guided Installation — 1,279 AED](https://buy.stripe.com/28EfZj3FC7p82HV2Po5J601)** — everything above + 1:1 setup on your infrastructure.

---

## License

MIT — see [LICENSE](LICENSE)

---

Created by [@Niiks7777](https://github.com/Niiks7777)
