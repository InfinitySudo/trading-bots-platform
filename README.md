# Multi-Strategy Crypto Futures Trading System

> Real-time signal detection, multi-strategy execution, paper/real hybrid mode, genetic-algorithm parameter optimizer with walk-forward validation, and a data-driven MFE calibration tool. Trades live capital on Bybit USDT-perp.

**Stack:** Python 3.11 · FastAPI · SQLite · WebSockets · NumPy/Pandas · systemd · Bybit V5 API · Telegram Bot API · Claude API (companion agent)

> This is a **showcase repository** describing the architecture and engineering decisions of a private production codebase. Source code is not included.

---

## What it does

A four-bot system that ingests live kline streams from 426 USDT-perpetual symbols, detects volume-spike signals, routes them through three strategies (CONSERVATIVE / TREND / AGGRESSIVE), and executes either as paper trades or real orders on Bybit — selectable per-strategy in production.

```
SignalBot ──► TradingBot ──► OrderExecutor ──┬──► PaperSim (CONS while GA tunes)
   │              │                          └──► Bybit V5 REST (TREND, AGGR)
   │              │
   └──── ControlBot (Telegram management) ───┘
              │
              ▼
        SmartBot (strategy switcher)
```

A separate companion agent (`gerchik-trading-agent`) implements Alexander Gerchik's price-action methodology (key levels, RR ≥ 3, money management) with a self-learning loop: rule engine + ML scorer + LLM reflector. Runs on a separate sub-account, shared symbol pool.

## Engineering highlights

### Genetic-algorithm parameter optimizer
- **27 genes** across CONS/TREND/AGGR strategies (entry thresholds, TP/SL ratios, trail params, position sizing).
- **Walk-forward validation:** 70/30 train/test split inside `evaluate()`. Fitness penalizes overfit: `fitness = test − K · saturating(train − test)`.
- **Overfit guards:** `MIN_TRADES_REQUIRED=50`, `MIN_TRADES_FULL=200`, `MIN_SYMBOL_COVERAGE=0.15`. Single-symbol moonshots get filtered.
- **Apply chain** writes optimized params to three places atomically: signal config JSON, `bot_settings` table, `strategy_parameters` table. Rollback supported via UI.
- **Subprocess discipline:** GA runs as `systemd-run --unit --slice` so cgroup-kill doesn't reap a 4-hour optimization mid-flight (`start_new_session=True` was not enough).

### MFE Calibration — replacing GA with empiricism
A second tuning approach that bypasses GA's overfit risk entirely: harvest real `peak_pnl_pct` (max favorable excursion) from completed trades, surface the histogram per strategy, and let the operator set TP caps to actual achieved highs rather than backtest-optimal ones. Shipped May 2026 after GA started producing unrealistic 13–48R take-profits while real MFE rarely exceeded 2R.

### Hybrid paper/real mode
Per-strategy routing in `OrderExecutorWrapper` — CONS trades stay in the simulator while GA is still tuning, TREND/AGGR hit Bybit. One position can never block another across modes (dup-symbol check filters by `pos['mode']`).

### Trading State soft-gate
A control surface that exposes four states (LIVE / GATED / PAUSED / DISABLED) with operator-driven transitions and a scorecard that recommends a state based on rolling drawdown, win-rate, and capital utilization. Replaced the hard kill-switch that required SSH.

### PnL accounting that doesn't lie
- **Money-based win rate:** `(realized_pnl − fees) > 0`. Fees were silently dropped for months — backfill script + `fees_paid_usd` persistence everywhere.
- **Source of truth for real PnL:** `get_bybit_realized_pnl()`, not the local DB. DB has orphaned rows (BE-fail in wrapper sometimes ate inserts) and chunk dups. Headline metrics query Bybit `closedPnl + openFee + closeFee`; partial closes split into N entries.
- **Realized vs. gross:** `realized_pnl_usd` = total across chunks; `gross_pnl_usd` = final chunk only. Documented because the original schema was ambiguous.

### Real-time infra
- WebSocket kline subscription to 426 symbols, ATR + buy/sell volume ratio confirmation.
- WS keepalive lesson learned the hard way: `run_forever()` without `ping_interval` → CLOSE_WAIT after 30s. Always `ping_interval=20, ping_timeout=10`.
- Telegram ControlBot is the kill switch — SIGTERM to ControlBot cascades to all trading bots (designed in: shutdown hook reaps the pool).
- Dashboard V2 with 6-tab UI (Stats / Charts / Control / GA / Symbols / Settings); legacy V1 still live for backwards compat.

### Backtest discipline
27-gene GA found `spike=6` returns +42% on BTC/ETH but median −13% across top-100 — edge is uneven by symbol. Documented and not glossed over. Symbols universe stratified into 3 tiers (426 active, 539 archived for backtest).

### Companion AI agent
`gerchik-trading-agent` is a separate process that:
- Identifies key support/resistance levels using Alexander Gerchik's price-action rules.
- Scores setups with a rule engine + lightweight ML classifier.
- Generates Telegram notifications with estimated RR, win-rate, and HTF reasoning.
- Uses an LLM reflector (Claude) to post-mortem closed trades and adjust the rule engine over time.
- Includes funding-rate guard + BTC correlation + volume context in reasoning.

## Scale

- ~40 production Python modules in `src/`.
- 4 systemd services, 1 dashboard, 1 GA worker pool, 1 Telegram control surface, 1 companion AI agent on a separate sub-account.
- Live wallet, live trades, real losses and wins both. Not a paper-only project.

## What's hard about it

Anyone can paper-trade a strategy that looks good on a 30-day backtest. The hard parts are:

1. **Detecting silent failures.** The DB lied about real PnL for weeks — orphan rows + duplicate chunks. Cross-source reconciliation against the exchange API is the only ground truth.
2. **GA without overfit.** A naive GA on a 27-gene space will find a single-symbol fluke. Walk-forward + coverage gates + fitness penalties are the difference between $5k delta and $5k drawdown. And sometimes (lesson learned May 2026) the right answer is to retire the GA in favor of empirical MFE calibration.
3. **Process supervision.** Long Popen jobs get SIGKILLed by cgroup cleanup if you don't put them in their own systemd slice. Lost a 4-hour run before figuring this out.
4. **Schema drift across paper/real tables.** `real_trades` is narrower than `simulated_trades`; `get_alltime_stats` crashes on missing columns if you don't UNION carefully.
5. **Order/signing bugs.** `_request("GET")` must sign the *same* querystring it sends; `sorted()` ≠ insertion order broke Bybit auth for a day.

## What I'd build next

- DEX migration (Hyperliquid candidate) — CEX custody risk is the biggest open question for any retail trader.
- Risk governor that auto-pauses on DD breach without human intervention.
- MT5 adapter layer to repurpose the strategy engine for prop-firm challenges (after 30+ real trades + 3-month +5% net history requirement is hit).

---

## About this repo

This is a **portfolio showcase**. The production code, API keys, and trading parameters are private. If you're a hiring manager or recruiter and want a deeper technical walkthrough — happy to do a live call.

**Author:** Artem Borysiuk — solo founder + retail trader building three production systems with AI as a force multiplier.

Live capital. Real edges. Real losses. No paper-only theater.
