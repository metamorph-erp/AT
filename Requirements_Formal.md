# Automated Quantitative Trading System — Requirements Document

**Project Name:** Auto-Trader (AT)  
**Version:** 3.7  
**Date:** March 4, 2026  
**Platform:** BSE India  
**Hosting:** Hetzner Cloud — CPX32 (AMD x86, 4 vCPU, 8 GB RAM, 160 GB NVMe)  
**ML Framework:** Qlib + LightGBM  
**Execution Layer:** OpenAlgo  

---

## 1. Objective

Build a fully automated quantitative trading system that collects market data, trains a machine learning model, generates trading signals, enforces mathematical risk controls, and executes trades on BSE India — operating autonomously as a daily closed-loop pipeline on an 8 GB VPS without any paid API dependencies.

### Capital Definition

**Total capital** (used throughout this document for guardrail calculations) is defined as the **current portfolio value: cash balance + market value of all open holdings at the latest available price**. This value is recomputed at every checkpoint and every evening batch. As the portfolio value changes, all percentage-based guardrails (G-001, G-003, G-004, S-009 ramp ceilings) scale proportionally — providing automatic de-risking during drawdowns and natural scaling during gains.

---

## 2. Design Philosophy

The system is not a conversational AI trading bot. It is a **small automated research lab** that runs a deterministic scientific loop every trading day:

1. **Observe** — Collect fresh market data
2. **Learn** — Update the statistical model
3. **Decide** — Rank stocks by predicted returns
4. **Protect** — Filter decisions through strict mathematical risk rules
5. **Act** — Execute a handful of controlled trades

All trading decisions are driven by machine learning models and deterministic rules — not by language models. This ensures millisecond execution, minimal RAM usage, and fully predictable behavior on a small server.

> *Boring systems are reliable systems. Markets reward consistency and risk control, not theatrical intelligence.*

### Why This Architecture

| Design Choice | Rationale |
|---------------|-----------|
| LightGBM over LLMs | Deterministic, millisecond inference, zero API cost, ideal for tabular financial data |
| Qlib for data + optimization | Battle-tested quant platform (Microsoft), handles data pipeline and portfolio math |
| OpenAlgo for execution | Broker-agnostic, supports Zerodha / Angel One / Upstox — swap brokers without code changes |
| Rules-based risk engine | No model hallucination risk; hard math enforces discipline |
| Daily batch cycle | Simpler, more reliable, and sufficient for equity swing trading on ≤50 stocks |

---

## 3. System Architecture

The system follows a closed-loop quantitative pipeline. There are two distinct flows: the **Evening Batch** (post-close, generates a plan) and the **Intraday Checkpoints** (market hours, executes that plan with live-price refreshes).

### Evening Batch Pipeline (runs post-close on trading days)

```
Market Data Sources (BSE Bhavcopy / Indian Stock Market API)
        ↓
  Data Collector (Python)
        ↓
  Qlib Dataset (structured storage)
        ↓
  Feature Engineering (momentum, volatility, RSI, MACD, etc.)
        ↓
  LightGBM Prediction (alpha score per stock)
        ↓
  Qlib Portfolio Optimizer (Mean-Variance Optimization)
        ↓
  Risk Manager (Python rules engine)
        ↓
  Baseline Plan → persisted to disk
        ↓
  Audit Log + Telegram Daily Report
```

> *The evening batch produces a plan — it does NOT execute orders. All order execution happens at the three intraday checkpoints the following trading day (see diagram below).*

### Component Roles

| Component | Role | Analogy |
|-----------|------|---------|
| LightGBM | Predicts expected stock returns from engineered features | **The Brain** — thinks |
| Qlib | Data pipeline + portfolio optimization (Mean-Variance) | **The Calculator** — optimizes |
| Risk Manager | Enforces position sizing, drawdown limits, VIX regime filters | **The Discipline** — protects |
| OpenAlgo | Routes approved orders to broker APIs | **The Hands** — acts |
| Scheduler (cron) | Triggers the daily cycle reliably | **The Clock** — orchestrates |

> *LightGBM thinks. Qlib calculates. Python enforces discipline. OpenAlgo acts.*

### Intraday Checkpoint Loop (3× per Trading Day)

Three active decision windows run every trading day, independent of the evening batch pipeline:

```
Indian Stock Market API (partial intraday OHLCV bar — best available at checkpoint time)
        ↓
  LightGBM Inference (current model artifact — not a retrain)  ← Checkpoint 1 only
        ↓
  Stock Re-ranking (refreshed alpha scores)  ← Checkpoint 1 only
        ↓
  Trade Proposal Generation (top-N buys / ranking-based sells / stop-loss sells)  ← Checkpoint 1 only
        ↓
  Risk Manager (all guardrails enforced: G-001 through G-013)
        ↓
  ┌─────────────────────────────────────────────────────────┐
  │  Sandbox:    OpenAlgo mock orders (no real broker)     │
  │  Trial:      Telegram HITL → approve/reject (15 min)   │
  │              Stop-loss timeout → auto-execute           │
  │  Production: execute via OpenAlgo (limit orders)       │
  └─────────────────────────────────────────────────────────┘
        ↓
  Broker API (Trial / Production only) → Audit Log
```

> *Checkpoints run at 9:30 AM, 11:00 AM, and 2:00 PM IST. The evening batch generates the overnight baseline plan; checkpoints replace it with live-price decisions during market hours.*
> **Checkpoint 1 (9:30 AM) runs the full pipeline above.** Checkpoints 2 & 3 (11:00 AM, 2:00 PM) run a lightweight subset: price + VIX fetch only → stop-loss checks → drawdown checks → VIX regime check. No inference, no re-ranking, no new buy proposals at checkpoints 2 & 3.

---

## 4. Pipeline Stages (Functional Requirements)

### 4.1 Data Ingestion

| ID     | Requirement |
|--------|-------------|
| D-001  | Collect daily OHLCV data (Open, High, Low, Close, Volume) for the active watchlist (up to 50 stocks) |
| D-002  | Data sources (all phases): BSE Bhavcopy (end-of-day, free), **Indian Stock Market API** (GitHub — real-time BSE/NSE price data, free, no API key required) |
| D-003  | Store raw data in local persistent storage on the VPS |
| D-004  | Data collection runs daily after market close (post 3:30 PM IST) |
| D-005  | Validate data integrity; reject and flag any corrupt or incomplete records |
| D-005a | **Intraday data validation:** at each checkpoint, validate Indian Stock Market API responses before use: **(a)** reject prices that are zero, negative, or unchanged from the previous trading day's close (stale data indicator); **(b)** reject prices that deviate >20% from the previous close (likely data error unless a circuit-breaker event is confirmed via G-009); **(c)** if >30% of watchlist stocks fail validation at a checkpoint, treat the entire checkpoint's price data as unreliable and skip the checkpoint (same behavior as E-008) |
| D-006  | Collect India VIX data daily for market regime detection; source: **NSE India** (India VIX is an NSE product — published at nseindia.com); verify that Indian Stock Market API (GitHub) surfaces this before development begins; if not, fetch directly from the NSE India public endpoint; India VIX is additionally fetched at each intraday checkpoint (IM-003) for real-time regime monitoring during market hours |
| D-007  | Collect Sensex index data daily (via Indian Stock Market API) as the relative strength benchmark for feature engineering |
| D-008  | Maintain a BSE trading calendar (annual holiday list); the scheduler uses this to skip non-trading days; refresh at the start of each calendar year |
| D-009  | This stage is intentionally simple — boring ingestion is reliable ingestion |

### 4.2 Data Preparation (Qlib)

| ID     | Requirement |
|--------|-------------|
| Q-001  | Use Microsoft Qlib to organize historical price data into structured datasets |
| Q-002  | Align all data by trading date; handle market holidays and missing data gracefully |
| Q-003  | Build rolling training windows for model retraining |
| Q-004  | Maintain at minimum **2 years** of historical data for model training (consistent with Q-006 bootstrap requirement and Assumption 16; 1 year is the absolute floor for inference but the model performs meaningfully better with 2 years of training history) |
| Q-005  | Qlib acts as the laboratory bench — messy market measurements become structured datasets ready for modeling |
| Q-006  | **Historical data bootstrap (Day 1):** before the first pipeline run, perform a one-time bulk download of at least 2 years of OHLCV history for all seed watchlist stocks from BSE Bhavcopy archives; this bootstrap step is a prerequisite for model training and must be completed before the system is run in Sandbox mode |

### 4.3 Feature Engineering

Raw prices alone are rarely enough. They must be transformed into statistical signals describing market behavior.

| ID     | Requirement |
|--------|-------------|
| F-001  | Generate technical features from raw price data including: **momentum, volatility, RSI, MACD, volume spikes, moving averages, relative strength vs Sensex index (^BSESN)**; Nifty is an NSE index and is out of scope — Sensex is the sole benchmark |
| F-002  | Produce a feature matrix (stocks × features × dates) as input for the ML model |
| F-003  | All features are computed deterministically — no randomness or external API calls |
| F-004  | Feature set is configurable and extensible without changing the core pipeline |
| F-005  | In scientific terms, these are the observables the model studies |

### 4.4 Model Training & Prediction (LightGBM)

LightGBM is a gradient boosting algorithm designed for fast tabular learning — ideal for financial feature matrices.

| ID     | Requirement |
|--------|-------------|
| M-001  | Use LightGBM as the primary prediction model |
| M-002  | Predict **expected return (alpha score)** for each stock over the next N trading days (configurable, default: 5 days) |
| M-003  | Retrain the model on a configurable schedule (default: **weekly full retrain on Sundays**); between retrains, the existing model artifact is used as-is for daily predictions — LightGBM is a batch algorithm and does not support incremental or online learning |
| M-004  | Model training and inference must complete within minutes on an 8 GB VPS |
| M-005  | Persist trained model artifacts to disk for crash recovery |
| M-006  | Log model performance metrics (prediction accuracy, portfolio Sharpe ratio) for ongoing monitoring |
| M-007  | If prediction produces anomalous outputs (extreme alpha scores), discard the batch and log the anomaly |
| M-008  | **Minimum holding period:** a stock bought at Checkpoint 1 must be held for a minimum of **3 trading days** before it becomes eligible for a ranking-based sell (S-007b); this prevents the model's 5-day prediction horizon from being wasted by next-day re-ranking churn; stop-loss exits (S-007a / G-006) are exempt — they can trigger at any time regardless of holding period |

### 4.5 Stock Ranking & Portfolio Optimization

Predictions alone are not trading decisions. They must be converted into optimized, risk-aware portfolio allocations.

| ID     | Requirement |
|--------|-------------|
| S-001  | Rank stocks by predicted alpha score (highest expected return first) |
| S-002  | Select top-N candidates for trading (configurable, default: top 5); top-N must always be ≤ G-002 (max simultaneous open positions, default 10) — at the default of top-5, G-002 is never the binding constraint; when all G-002 position slots are occupied, no new buys are triggered regardless of ranking |
| S-003  | Use **Qlib's portfolio optimization (Mean-Variance Optimization)** to determine optimal position weights |
| S-004  | Optimizer considers: individual stock volatility, cross-stock correlations, total portfolio risk exposure |
| S-005  | Output: a list of trade actions (buy/sell) with specific share quantities per stock |
| S-006  | The optimizer converts raw alpha scores into capital allocation — "how much to buy" is as important as "what to buy" |
| S-007  | **Position exit rules (combination approach):** (a) **Stop-loss exit** — if a held position’s unrealised loss reaches the trailing stop-loss floor (G-006 — 5% below highest-seen price since entry), the Intraday Trading Checkpoint submits a market sell order immediately via OpenAlgo; (b) **Ranking-based exit** — if a held stock drops out of the top-N ranked stocks at **Checkpoint 1**, a ranking-based sell proposal is generated at Checkpoint 1 (re-ranking only runs at CP1 — see IM-005); (c) Stop-loss always takes precedence — a position already flagged for stop-loss exit is not retained by a re-ranking result |
| S-008  | A stock already held in the portfolio is re-ranked each daily cycle; if it remains in the top-N, no new buy order is generated — the existing position is retained as-is |
| S-009  | **Cold-start / zero-holdings ramp:** whenever the system starts a new mode with zero open positions (initial Sandbox deployment **or any mode transition** — e.g., Trial → Production where the Production portfolio starts fresh), capital is deployed on a fixed 5-day ramp to avoid large initial exposure: Day 1 — max 20% of capital, Day 2 — max 40%, Day 3 — max 60%, Day 4 — max 80%, Day 5 onward — full capital available; at the Trial → Production transition the full ₹50,000 Production allocation is subject to the ramp from day 1; the ramp applies to the total deployed capital ceiling, not to individual position limits (G-001/G-003 still apply on each day) |
| S-010  | **G-005 Tier 2 recovery path:** after a full portfolio liquidation triggered by G-005 Tier 2 (15% drawdown), the system enters a **cooling-off period** of **5 trading days** during which no new buy orders are permitted; **during cooling-off, the evening batch continues to run normally** — data is collected, features are updated, and a baseline plan is generated; however, the plan is flagged as **non-executable** and no proposals are forwarded to checkpoints; this keeps data and model artifacts current for re-entry; **at the end of the cooling-off period, the all-time portfolio peak is reset to the current cash balance** — this prevents a deadlock where the old peak permanently blocks re-entry; the S-009 cold-start ramp is then automatically triggered (Day 1: 20%, Day 2: 40%, etc.) to gradually re-enter the market; Telegram notification is sent at the start of cooling-off, at peak reset, and at ramp completion; **no human intervention is required in Production**; in Trial mode, a Telegram approval is required before the ramp begins |

### 4.6 Risk Manager

See §6 for detailed risk guardrails. The risk manager is a pure Python rules engine — no ML, no API calls, millisecond execution.

### 4.7 Trade Execution (OpenAlgo)

| ID     | Requirement |
|--------|-------------|
| X-001  | Use **OpenAlgo** as the broker-agnostic execution layer |
| X-002  | OpenAlgo connects to supported Indian brokers: Zerodha, Angel One, Upstox, and others |
| X-003  | In Sandbox mode, OpenAlgo is used in its built-in **mock/paper order mode** — orders are submitted to the locally hosted OpenAlgo instance but are not forwarded to any real broker; order fills are simulated by OpenAlgo at the fetched real-time price |
| X-004  | Log every order sent, including status (filled, rejected, partial fill) |
| X-005  | Trades are executed as **limit orders** at the last-traded-price ± a configurable buffer (default: **0.3%**); buy orders use LTP + buffer, sell orders use LTP − buffer; **exception:** stop-loss exits (G-006 / IM-007) are submitted as **market orders** to guarantee execution — capital protection takes precedence over fill quality; new buy/sell proposals are generated and executed at **Checkpoint 1 (9:30 AM)** only; stop-loss market sells (IM-007) may execute at any checkpoint (9:30 AM, 11:00 AM, or 2:00 PM IST); the 9:30 AM checkpoint is the primary execution window for the evening batch baseline plan; **unfilled limit order lifecycle:** limit orders submitted at CP1 that remain unfilled are **auto-cancelled** at the next checkpoint (CP2/CP3) per IM-006a; no re-submission occurs at CP2/CP3 — new proposals are only generated at CP1 |
| X-006  | Before executing a sell order, validate that the portfolio holds that stock; discard and log if not held |
| X-007  | Execution must be reliable and predictable — no retries on partial fills unless explicitly configured |
| X-008  | **Broker session management:** before the 9:10 AM pre-market preparation, a cron job at **9:00 AM IST** automatically refreshes the broker API session token using the stored TOTP secret; the TOTP secret is stored **encrypted on disk** using a machine-specific key; if token refresh fails, retry up to 3 times with 30-second intervals; if all retries fail, send a Telegram alert and halt all trading for the day; the session token is valid for one trading day; **Phase 1 (Sandbox):** this step is skipped — OpenAlgo mock mode does not require broker authentication |
| X-009  | **Liquidity validation:** before submitting any buy order, verify that the stock's **BSE-specific** average daily traded value (last 20 trading days) exceeds a configurable minimum (default: **₹50 lakh / day**); combined BSE+NSE volume must NOT be used — many stocks trade primarily on NSE and have thin BSE volumes; if below threshold, the trade proposal is skipped and logged as illiquid; this prevents poor fills on thinly-traded BSE stocks |

### 4.8 Reporting Pipeline (Daily)

| ID     | Requirement |
|--------|-------------|
| RP-001 | Generate a daily portfolio snapshot: holdings, quantities, current market values |
| RP-002 | Calculate and report current profit/loss (with estimated tax impact) |
| RP-003 | Report model prediction accuracy and cumulative performance metrics |
| RP-004 | Log every trade decision: executed, skipped, or blocked by guardrails — with full reasoning |
| RP-005 | Maintain a full audit trail / transaction log accessible at any time |
| RP-005a | **Decision traceability:** every trade execution log entry must record: **(a)** model artifact version (file hash or timestamp); **(b)** input feature snapshot (alpha score, top features contributing to the signal); **(c)** guardrails evaluated and their pass/fail status; **(d)** portfolio state at decision time; this enables reproducing the exact decision path for any historical trade |
| RP-006 | Send daily summary report to Telegram after market close, on trading days only (all modes) |
| RP-007 | Include risk guardrail activation events in daily report |
| RP-007a | **Daily heartbeat:** send a Telegram message at **9:15 AM IST on trading days** ("AT alive — pre-market starting") and at **12:00 PM IST on weekends/holidays** ("AT alive — idle"); this provides instant detection if the VPS goes offline — a missing heartbeat is the earliest signal of system unavailability; heartbeat messages are brief (one line) to avoid alert fatigue |
| RP-008 | At the end of each evening batch, after stock ranking, persist the current day's **top-N stock membership list** to disk — a simple dated record of which stocks ranked in the top-N that day; the Saturday screener (W-003/W-005) reads the last 5 weeks of this history to identify stocks absent from top-N for 4 or more consecutive weeks; this file is append-only and must survive VPS restarts |

---

### 4.9 Intraday Trading Checkpoints

Three structured trading decision windows run every trading day. They operate in two tiers:

- **Checkpoint 1 (9:30 AM) — Full Decision Checkpoint:** fetches live prices and VIX, runs LightGBM inference, re-ranks all watchlist stocks, generates buy and ranking-based sell proposals, applies all guardrails, and executes orders (or sends HITL in Trial mode).
- **Checkpoints 2 & 3 (11:00 AM and 2:00 PM) — P&L Monitoring Checkpoints:** fetch live prices and VIX only; evaluate stop-loss exits, daily loss limit, total drawdown, and VIX regime change. No LightGBM inference, no re-ranking, no new buy proposals.

This two-tier design retains full mid-session risk protection while eliminating the noise and complexity of running ML inference on incomplete 90-minute-old intraday bars.

> **Accepted trade-off:** stop-losses and risk limits are checked at checkpoint times only. Events between checkpoints are not caught until the next check. Maximum unmonitored window is ~90 minutes. This is an intentional simplicity choice.

| ID     | Requirement |
|--------|-------------|
| IM-001 | Three intraday checkpoints run on every trading day: **9:30 AM, 11:00 AM, and 2:00 PM** IST. They operate in two tiers: **(a) Checkpoint 1 (9:30 AM) is the Full Decision Checkpoint** — runs the complete cycle IM-003 through IM-013; **(b) Checkpoints 2 & 3 (11:00 AM and 2:00 PM) are P&L Monitoring Checkpoints** — run IM-003 (price + VIX fetch), **IM-006a (unfilled order cleanup)**, IM-007 (stop-loss), IM-008 (daily P&L), IM-009 (total drawdown), IM-010 (VIX regime), IM-011 (3:30 PM cancellation), IM-012 (independence), IM-013 (logging) only; IM-004, IM-005, and IM-006 do not run at Checkpoints 2 & 3 |
| IM-002 | **Pre-market preparation (9:10 AM, before Checkpoint 1):** re-fetch India VIX; apply G-007/G-008 regime filters to the baseline plan from the previous evening's batch; this is not a trading checkpoint — no proposals are generated and no orders are placed |
| IM-003 | At each checkpoint, fetch two data items: **(a)** the **partial intraday OHLCV bar** for **all watchlist stocks** (up to 50) via **Indian Stock Market API** (GitHub) — current session open, session high-so-far, session low-so-far, last traded price as close proxy, and session volume accrued so far; **(b)** the current **India VIX level** from the NSE India endpoint (or Indian Stock Market API if confirmed to surface it) — used immediately in IM-010 to detect any regime threshold crossing since the previous checkpoint |
| IM-004 | **(Checkpoint 1 only)** Using the partial intraday OHLCV bar fetched in IM-003, run a quick inference pass with the current LightGBM model artifact to refresh alpha scores for all watchlist stocks; this is **not a retrain** — it uses the model persisted from the last weekly Sunday retrain; **performance note:** feature engineering at the checkpoint uses an incremental update — the new intraday bar is substituted into the last row of the existing in-memory feature matrix; the full 2-year history is NOT re-loaded or re-computed at checkpoint time |
| IM-005 | **(Checkpoint 1 only)** Re-rank all watchlist stocks by refreshed alpha scores; generate updated trade proposals (buys for top-ranked opportunities, ranking-based sells for positions that have dropped out of the top-N) subject to all guardrails (G-001 through G-013) and current open portfolio state; **position sizing uses the MVO weights from the previous evening batch — MVO (S-003) is NOT re-run at checkpoints** |
| IM-006 | **(Checkpoint 1 only) Trade execution — (a) Production / Sandbox:** execute approved proposals as **limit orders** (per X-005: LTP ± 0.3% buffer) via OpenAlgo; in Sandbox mode OpenAlgo operates in **mock/paper order mode** — orders processed by OpenAlgo but not forwarded to any real broker; fills simulated at fetched real-time price; unfilled limit orders from CP1 are auto-cancelled at the next checkpoint per IM-006a (no re-submission at CP2/CP3); **(b) Trial mode:** send HITL proposals to Telegram with full trade details (stock, direction, quantity, current price, alpha score, guardrail status); human has **15 minutes** to approve/reject; any proposal not responded to within 15 minutes is **auto-cancelled** |
| IM-006a | **(Checkpoints 2 & 3) Unfilled order cleanup:** at each P&L monitoring checkpoint, query OpenAlgo for any unfilled or partially-filled limit orders submitted at Checkpoint 1; **auto-cancel** all unfilled orders — the original LTP-based limit price is stale; log each cancellation with the reason (unfilled after ~90 minutes); partially-filled orders: the filled portion is retained; the unfilled remainder is cancelled; no re-submission occurs at CP2/CP3 — new buy proposals are only generated at CP1 |
| IM-007 | At each checkpoint, check all open positions for stop-loss floor (G-006); if triggered: **(a) Production / Sandbox:** submit market sell order immediately via OpenAlgo; **(b) Trial mode:** send HITL request to Telegram; 15-minute window; **auto-executes** if no response (capital protection takes precedence over HITL timeout) |
| IM-008 | At each checkpoint, compute cumulative portfolio P&L for the day; if G-004 (3% daily loss) is breached, cancel all pending buy proposals, halt all new buy orders for the remainder of that trading day, and send a Telegram alert; **sell orders and stop-loss exits remain active** |
| IM-009 | At each checkpoint, compute total portfolio drawdown from the all-time portfolio peak (not just that day); **(a)** if **G-005 Tier 1** (10% total drawdown) is breached, halt all new buy orders; send Telegram alert; set a persistent **Tier 1 active** flag; **(b)** if the Tier 1 active flag is set from a previous checkpoint or previous day, check whether drawdown has recovered below the hysteresis threshold (Tier 1 threshold − 2 pp); if recovered, clear the flag, resume buying, and send a Telegram notification; **(c)** if **G-005 Tier 2** (15% total drawdown) is breached, submit **market sell orders for all tradeable open positions** via OpenAlgo — **frozen holdings** (W-007) are excluded since they cannot be traded (consistent with G-005); **G-005 Tier 2 explicitly overrides G-004** — liquidation sell orders are submitted even if G-004 has already halted new buys |
| IM-010 | Using the India VIX level fetched in IM-003, check whether VIX has crossed a new G-007/G-008 regime threshold since the previous checkpoint (or since the 9:10 AM pre-market fetch for Checkpoint 1); if a new threshold has been breached, apply the updated position sizing restrictions to all proposals at this checkpoint and send a Telegram alert; **VIX recovery (VIX dropping below a threshold) does not restore position sizes during the same trading day** — the next evening's batch generates a fresh full-size plan; the conservative halved plan stands for the current day |
| IM-011 | At **3:30 PM** (market close), cancel all remaining HITL-awaiting **buy/sell proposals (Checkpoint 1 proposals only)** that have not received Telegram approval; stop-loss HITL requests are not cancelled here — they auto-execute via the 15-minute timeout defined in IM-007 and will already be resolved well before 3:30 PM; log all cancellations; the checkpoint process stops for the day |
| IM-012 | The intraday checkpoint process and the evening batch pipeline are independent processes; a crash in one must not affect the other |
| IM-013 | Log every checkpoint cycle with full detail: timestamp, checkpoint tier (full decision or P&L monitoring), prices fetched, alpha scores refreshed (CP1 only), proposals generated (CP1 only), guardrails triggered, orders executed / simulated / sent for HITL / cancelled |


---

### 4.10 Watchlist Management

The watchlist tracks up to 50 stocks the system researches and trades. In **Trial/Sandbox**, rule-based screening generates candidates and a human confirms all additions and removals via Telegram. In **Production**, watchlist changes are auto-applied and the user receives a notification.

| ID     | Requirement |
|--------|-------------|
| W-001  | Maximum watchlist size: **50 stocks** at any time (consistent with G-011) |
| W-002  | The system starts with a manually configured seed list |
| W-003  | A weekly rule-based screener runs as an **independent cron job every Saturday** — completely separate from trading-day processes. It performs two tasks: **(a) Additions** — evaluates stocks in the **user-maintained candidate pool** (a manually curated list of 50–150 pre-approved stocks, maintained separately from the active watchlist); addition criterion is a single lightweight check: last-close traded volume above a configurable threshold; **data source for candidate pool volume:** the screener fetches Friday's closing volume for candidate pool stocks from **Indian Stock Market API** (or BSE Bhavcopy if the API doesn't cover that stock) — this is a small one-off fetch of ≤150 volume values, not a full OHLCV ingestion; this avoids fetching data for hundreds of unknown stocks; **(b) Removals** — evaluates W-005 removal candidates using the top-N membership history persisted by the evening batch (RP-008); candidate additions and removals are consolidated into a single Telegram message — **Trial/Sandbox:** sent for human approval; **Production:** auto-applied with notification (see W-004); **note:** scanning the full BSE 500 universe is deferred to a future phase — the current design keeps the screener fast, focused, and implementable without external data dependencies |
| W-004  | **Trial/Sandbox:** Candidate additions and removals are sent to the user via **Telegram** for review — the system never modifies the watchlist autonomously (except W-007 auto-removal); **Production:** watchlist changes from the Saturday screener are **auto-applied** and a Telegram **notification** (not approval request) is sent; W-007 auto-removal applies in all modes |
| W-005  | A stock is flagged as a removal candidate when it has not appeared in the top-N ranked signals for the past **4 consecutive weeks** (configurable); this is evaluated by the **Saturday screener (W-003)** by reading the daily top-N membership history persisted by the evening batch (RP-008); the screener checks the last 5 weeks of records and flags any stock absent for 4 or more consecutive weeks; flagged stocks are included in the Saturday Telegram approval message alongside addition candidates |
| W-006  | **Trial/Sandbox:** Human approves or rejects each watchlist change via Telegram within **24 hours**; if no response, the change is **not applied**; **Production:** watchlist changes are auto-applied (see W-004) |
| W-007  | Stocks suspended, delisted, or circuit-breaker-halted by BSE for more than 3 consecutive trading days are automatically removed from the watchlist and flagged in the daily report; **if the removed stock has an open position:** the position remains in the portfolio as a **frozen holding** — it is excluded from re-ranking (S-008), not counted toward G-002 (max open positions), and not eligible for new buy orders; the stop-loss check (IM-007) continues to run against it at each checkpoint using the last available price; when the stock resumes trading, the system submits a **market sell order** at the next checkpoint to liquidate the frozen position; a Telegram alert is sent on freeze, on resume, and on liquidation |
| W-008  | All watchlist changes (additions, removals, rejections) are logged with timestamp and reason |
| W-009  | **New stock historical data (on addition):** upon watchlist addition — whether via human approval (Trial/Sandbox per W-006) or auto-apply (Production per W-004) — immediately trigger a background bulk download of at least **2 years of OHLCV history** for that stock from BSE Bhavcopy archives; load the data into the Qlib dataset; the stock is included in the **next Sunday full model retrain (M-003)**; the stock does not participate in trading proposals at checkpoints until its historical data load is complete and confirmed |

---

## 5. Daily Schedule

The system runs three types of scheduled processes on distinct cadences: intraday trading checkpoints (market hours), an evening batch pipeline (post-close), and weekly independent jobs (weekends).

### 5.1 Pre-Market Preparation (Trading Days Only)

| Time (IST) | Action |
|------------|--------|| 9:00 AM | **Broker session refresh (Phase 2 onwards):** auto-generate TOTP code from stored encrypted secret; authenticate with broker via OpenAlgo; obtain fresh API session token; retry up to 3× on failure; if all retries fail, halt trading for the day and send Telegram alert || 9:10 AM | Re-fetch India VIX; apply G-007/G-008 regime filters to the baseline plan from the previous evening's batch; **no orders placed** — this is preparation only |

### 5.2 Intraday Trading Checkpoints (Trading Days Only)

Three independent decision windows per day operating in two tiers (see §4.9 IM-001): **Checkpoint 1 (9:30 AM)** runs the full pipeline (inference + re-ranking + proposals + execution); **Checkpoints 2 & 3 (11:00 AM / 2:00 PM)** run P&L monitoring only (price + VIX fetch, stop-loss, drawdown, VIX regime).

| Time (IST) | Checkpoint | Summary |
|------------|------------|---------|
| 9:30 AM | **Checkpoint 1 — Full Decision** | Fetch live prices + VIX → LightGBM inference → Re-rank → Generate proposals → Apply all guardrails → Execute or HITL (Trial) |
| 11:00 AM | **Checkpoint 2 — P&L Monitor** | Cancel unfilled CP1 limit orders (IM-006a) → Fetch live prices + VIX → Stop-loss check → Daily P&L check (G-004) → Total drawdown check (G-005) → VIX regime check (G-007/G-008) → Execute stop-loss sells only |
| 2:00 PM | **Checkpoint 3 — P&L Monitor** | Same as Checkpoint 2 (no unfilled orders expected; cleanup runs as a no-op); any still-pending stop-loss HITL approvals at 3:30 PM are auto-executed |
| 3:30 PM | Market close | Cancel remaining HITL-awaiting proposals (CP1 only); auto-execute any pending stop-loss approvals; checkpoint process stops for the day |

**Non-trading days:** No checkpoint process runs (weekends, BSE holidays).

### 5.3 Evening Batch (Trading Days Only)

| Time (IST) | Action |
|------------|--------|
| 4:00 PM | Data collection — fetch EOD OHLCV for all watchlist stocks, India VIX closing level, Sensex closing level |
| 4:30 PM | Feature engineering + LightGBM **prediction** using current model artifact (weekly full retrain is a separate Sunday job — see §5.4) |
| 5:00 PM | Stock ranking + Qlib portfolio optimization (baseline plan for next day's checkpoints); **persist today's top-N membership list to disk** (RP-008) for use by the Saturday screener (W-005) |
| 5:15 PM | Persist baseline plan to disk (used by next morning's pre-market preparation at 9:10 AM); persist full system state for crash recovery |
| 5:30 PM | Daily report generation → Telegram summary |

**Non-trading days:** Evening batch does not run (weekends, BSE holidays).

### 5.4 Weekly Independent Jobs (Weekends)

These are separate cron jobs with no dependency on trading day processes.

| Day / Time (IST) | Job | Action |
|-----------------|-----|--------|
| Saturday 10:00 AM | Watchlist Screener | **(a)** Evaluates **user-maintained candidate pool** for addition candidates (volume-only criterion: last-close volume above threshold); **(b)** Evaluates W-005 removal candidates from top-N membership history (RP-008) |
| Saturday 10:30 AM | Watchlist Screener | Consolidated list of candidate additions and removals sent to user via Telegram — **Trial/Sandbox:** for approval; **Production:** auto-applied with notification (see W-004) |
| Sunday **2:00 AM IST** | Model Retrain | LightGBM weekly full retrain using the latest accumulated OHLCV dataset (M-003); 2:00 AM chosen because VPS is idle (no concurrent trading or screener workloads) and the retrain completes well before Monday's 9:10 AM pre-market preparation |

---

## 6. Risk Guardrails

All guardrails are **hard-coded** (not model-decided) and enforced before any trade execution. This layer is critical — many trading systems fail not because of bad predictions but because of bad risk control.

### 6.1 Position Sizing

| ID     | Guardrail |
|--------|-----------|
| G-001  | Maximum **total holding** per stock: **5% of total capital** (e.g., ₹2,500 of a ₹50,000 portfolio); configurable per mode |
| G-002  | Maximum simultaneous open positions: **10** (configurable per mode) |
| G-003  | Maximum **single new order** size: **3% of total capital** per transaction (e.g., ₹1,500); configurable per mode. Distinct from G-001: G-001 limits the resulting total holding; G-003 limits the size of each individual order |

### 6.2 Drawdown Protection

| ID     | Guardrail |
|--------|-----------|
| G-004  | Maximum daily loss limit: **3% of portfolio** — measured against the **previous trading day's closing portfolio value** (computed during the evening batch using EOD close prices from data ingestion at 4:00 PM; this value is persisted alongside the baseline plan and used as a fixed intraday reference the following day); halt all **new buy orders** for the remainder of that day if breached; alert via Telegram; resume automatically next trading day; sell orders and stop-loss exits remain active; **G-005 Tier 2 (15% total drawdown liquidation) explicitly overrides this halt** — liquidation sell orders are submitted regardless of G-004 state |
| G-005  | **Two-tier total portfolio drawdown protection:** **(a) Tier 1 — 10% drawdown from peak (default):** halt all new buy orders; existing positions are retained; sell orders and stop-loss exits remain active; send Telegram alert; system resumes buying automatically the **next trading day** if drawdown has recovered below **Tier 1 threshold minus 2 percentage points** (i.e., 8% for Production/Sandbox default of 10%, 6% for Trial default of 8%) — this hysteresis band prevents daily oscillation at the threshold; **(b) Tier 2 — 15% drawdown from peak (default):** submit **market sell orders for all tradeable open positions** via OpenAlgo — **frozen holdings** (W-007 — suspended/delisted stocks) are excluded from liquidation sell orders since they cannot be traded; the Telegram alert lists frozen holdings separately with their last known value; halt all new buys; send Telegram alert with full position details; configurable per mode; **G-005 Tier 2 explicitly overrides G-004** — liquidation orders are submitted even if G-004 has halted buys; in Trial mode, liquidation sell orders follow the HITL 15-minute approval window — if no response within 15 minutes, liquidation **auto-executes** (capital protection takes precedence, same as stop-loss auto-execute in IM-007); **peak reset:** after a Tier 2 liquidation, the all-time portfolio peak is reset per S-010 |
| G-006  | **Trailing stop-loss** on every open position: the stop-loss floor starts at **5% below entry price** (default) and **ratchets upward** as the stock price rises — the floor is always maintained at 5% below the **highest price observed since entry**, where the highest price is the **session high-so-far** value from IM-003 (not the LTP) — this captures the true intraday peak even if the price has retreated by checkpoint time; the floor never moves downward; configurable per mode (e.g., tighter in Trial mode); example: stock bought at ₹100 → floor = ₹95; stock rises to ₹130 (session high) → floor ratchets to ₹123.50; stock falls to ₹123 (LTP at checkpoint) → stop-loss triggered |

### 6.3 Market Regime Filters (India VIX)

| ID     | Guardrail |
|--------|-----------|
| G-007  | If **India VIX > 25** (extreme market panic): reduce all LightGBM-suggested position sizes by **50%** |
| G-008  | If **India VIX > 35** (crisis): halt all new buy orders; allow only sell/exit orders |
| G-009  | Skip trading (cancel all pending orders) during market-wide circuit breaker events; **detection:** at each **intraday trading checkpoint** (IM-001), compute the Sensex intraday change from the previous close using Indian Stock Market API; if Sensex is down ≥ **10% intraday**, treat as a circuit breaker event, halt all activity, and send a Telegram alert; **Trial/Sandbox:** resume only the following trading day after human confirmation via Telegram; **Production:** resume automatically the next trading day — Telegram receives a notification (not approval request); **note:** SEBI’s official circuit-breaker index is Nifty 50 (NSE), which is out of scope for this system; Sensex is used as an equivalent proxy — both indices are highly correlated and a 10% drop in Sensex reliably signals a market-wide halt |

### 6.4 Operational Limits

| ID     | Guardrail |
|--------|-----------|
| G-010  | Maximum number of **new buy/sell proposals** per day: **10** (Production default), **3** (Trial default — all generated at the 9:30 AM checkpoint), **unlimited** (Sandbox — for simulation completeness); configurable per mode; stop-loss sells triggered at Checkpoints 2 & 3 do not count toward this limit |
| G-011  | Maximum watchlist size: **50 stocks** at any time |
| G-012  | All guardrail parameters shall be configurable per mode (Sandbox / Trial / Production); **every configuration change must be logged** with: timestamp, parameter name, old value, new value, and the mode affected; this log is append-only and included in the audit trail (RP-005) to ensure full traceability — if a past trade decision is reviewed, the guardrail values in effect at that time can be reconstructed |
| G-013  | Send Telegram alert whenever any guardrail is triggered |

### 6.5 Guardrail Defaults by Mode

All guardrail values are configurable. Trial mode uses tighter defaults to protect real capital at small scale. Sandbox mirrors Production for realistic simulation.

| Guardrail | Parameter | Sandbox | Trial | Production |
|-----------|-----------|---------|-------|------------|
| G-001: Max position size per stock | % of total capital | 5% | 3% | 5% |
| G-002: Max simultaneous open positions | Count | 10 | 5 | 10 |
| G-003: Max single order size | % of total capital | 3% | 2% | 3% |
| G-004: Daily loss halt | % of portfolio | 3% | 2% | 3% |
| G-005 Tier 1: Drawdown buy-halt | % from portfolio peak | 10% | 8% | 10% |
| G-005 Tier 1: Resume threshold | % from portfolio peak | 8% | 6% | 8% |
| G-005 Tier 2: Drawdown liquidation | % from portfolio peak | 15% | 12% | 15% |
| G-006: Stop-loss floor per position | % below entry price | 5% | 3% | 5% |
| G-010: Max new transactions / day | Count | Unlimited | 3 (all at 9:30 AM) | 10 (all at 9:30 AM) |

> **Trial mode minimum order floor:** G-003 Trial default (2% of ₹10,000 = ₹200/order) is near Indian broker minimum transaction costs (~₹20/order flat = ~10% cost ratio). A minimum absolute order floor of **₹500** applies in Trial mode regardless of the percentage-based G-003 calculation; if the percentage yields less than ₹500, the trade proposal is skipped and logged as unviable.

---

## 7. Operating Modes

The system supports three sequential operating modes:

### 7.1 Sandbox Mode

| Attribute          | Detail |
|--------------------|--------|
| Duration           | Minimum 1 month |
| Funds              | ₹50,000 (simulated — mirrors Production fund for realistic testing) |
| Transactions       | Mock orders — submitted through **OpenAlgo's paper/mock order mode**; not forwarded to any real broker; fills simulated by OpenAlgo at real-time price |
| Broker / OpenAlgo  | **Integrated (mock mode)** — OpenAlgo is self-hosted on the VPS and running in paper/mock order mode; no real broker connection required |
| HITL               | **None** — fully autonomous |
| Data               | Real market data |
| Purpose            | Validate model quality, strategy profitability, and system stability |
| Exit Criteria      | Human reviews Sandbox performance and manually promotes to Trial |
| Advisory exit benchmarks | Minimum 22 trading days (1 calendar month); annualised Sharpe ratio ≥ 0.5; maximum realised drawdown < **10%** (staying within G-005 Tier 1; reaching Tier 2 during Sandbox indicates a fundamental strategy problem); no uncaught guardrail violations; these are advisory — final promotion decision is human judgement |

### 7.2 Trial Mode

| Attribute          | Detail |
|--------------------|--------|
| Duration           | Until human is satisfied with performance |
| Funds              | ₹10,000 (real money) |
| Transactions       | Real — executed via OpenAlgo → broker API |
| Broker / OpenAlgo  | **Integrated** with selected broker |
| HITL               | **Yes** — human approval required via **Telegram Bot** before each trade |
| HITL Workflow      | At **Checkpoint 1 (9:30 AM)** the system sends trade proposals to Telegram with full details (stock, direction, quantity, current price, alpha score, guardrail status); human has **15 minutes** to approve/reject each proposal; proposals not responded to within 15 minutes are **auto-cancelled**; Checkpoints 2 & 3 send only stop-loss alerts (which auto-execute on 15-minute timeout per IM-007) — no new buy proposals at those checkpoints |
| Stop-loss Exception | Stop-loss exits (IM-007) are checked at each checkpoint; in Trial mode a Telegram approval request is sent; human has **15 minutes** to approve/reject; if no response, the stop-loss **auto-executes** to protect real capital; capital protection takes precedence over HITL timeout at all times |
| Data               | Real market data |
| Purpose            | Prove the system works with real money on small scale |
| Exit Criteria      | Human reviews Trial performance and manually promotes to Production |

### 7.3 Production Mode

| Attribute          | Detail |
|--------------------|--------|
| Duration           | Ongoing |
| Funds              | ₹50,000 (real money, allocated) |
| Transactions       | Real — executed via OpenAlgo → broker API |
| Broker / OpenAlgo  | **Integrated** with selected broker; TOTP auto-login at 9:00 AM (X-008) |
| HITL               | **None** — fully autonomous |
| Watchlist           | Auto-applied additions/removals from Saturday screener (W-004); W-007 auto-removal; Telegram receives notifications, not approval requests |
| Circuit Breaker     | Auto-resume next trading day (G-009); Telegram notification only |
| Portfolio Reconciliation | Daily broker vs internal state check (P2-008) |
| Data               | Real market data |
| Purpose            | Full-scale autonomous trading |

---

## 8. Delivery Phases

### Phase 1 — Sandbox System (OpenAlgo Mock Mode — No Real Broker)

| ID     | Deliverable |
|--------|-------------|
| P1-001 | Data ingestion pipeline (BSE Bhavcopy + Indian Stock Market API + India VIX + Sensex index + BSE trading calendar) |
| P1-002 | Qlib data preparation and feature engineering |
| P1-003 | LightGBM model training and prediction pipeline |
| P1-004 | Stock ranking and Qlib portfolio optimization |
| P1-005 | Risk manager with all guardrails enforced (including VIX regime filters) |
| P1-006 | OpenAlgo deployment in **mock/paper order mode** — self-hosted on the VPS; orders flow through the full OpenAlgo execution path but are not forwarded to any real broker; this enables Phase 2 broker connection with zero code changes (swap mock mode for live broker credentials) |
| P1-007 | Reporting pipeline with daily Telegram summaries |
| P1-008 | Logging and full audit trail |
| P1-009 | Cron-based daily scheduler |
| P1-010 | Local development and testing first, then deploy to Hetzner |
| P1-011 | System runs in Sandbox mode for minimum 1 month |
| P1-012 | Intraday Trading Checkpoints — three daily decision windows (9:30 AM / 11:00 AM / 2:00 PM) in two tiers: **Checkpoint 1 (9:30 AM)** — LightGBM inference, signal refresh, proposal generation, full guardrail enforcement, and order execution via **OpenAlgo mock/paper mode**; **Checkpoints 2 & 3 (11:00 AM / 2:00 PM)** — P&L monitoring only (stop-loss, drawdown, VIX regime checks); no real broker in either tier |
| P1-013 | Watchlist Management — weekly Saturday screener, Telegram-based approval workflow, auto-removal logic for suspended/delisted stocks |
| P1-014 | BSE trading calendar integration — annual holiday list loaded into cron scheduler to skip non-trading days |
| P1-015 | Indian Stock Market API (GitHub) integration — real-time BSE/NSE data used by both daily batch pipeline and intraday checkpoints |
| P1-016 | Historical data bootstrap (Q-006) — one-time bulk download of 2 years of OHLCV from BSE Bhavcopy archives for all seed watchlist stocks; prerequisite for first model training run |
| P1-017 | **Backtesting framework:** before entering live Sandbox simulation, run the complete strategy (feature engineering → LightGBM prediction → ranking → MVO → guardrails → simulated execution) against the bootstrapped 2-year historical data; **must model realistic transaction costs:** brokerage (~₹20/order), STT (**0.1% on both buy and sell sides** — delivery/CNC rate, since M-008 enforces a 3-day minimum holding period making all trades delivery trades; the 0.025% intraday rate does NOT apply), stamp duty, and GST; apply a configurable **slippage estimate** (default: 0.1% per trade) to simulate real fill prices; produce a backtesting report with: annualised return (net of costs), Sharpe ratio, max drawdown, win rate, avg holding period, total transaction costs paid, and comparison vs buy-and-hold Sensex; this is a **one-time validation gate** — Sandbox mode does not begin until backtesting results meet minimum viability (Sharpe ≥ 0.3, max drawdown < 25%); backtesting can be re-run after any significant model or feature change |
| P1-018 | **systemd service configuration** for all system processes (see I-010); auto-start on boot; watchdog monitoring |
| P1-019 | **VPS hardening** per I-011 (SSH keys, firewall, credential encryption, log scrubbing) |
| P1-020 | **Startup self-diagnostic** (E-011) and **graceful shutdown** (E-010) |
| P1-021 | **Testing suite:** **(a)** unit tests for all guardrail rules (G-001 through G-013) with boundary conditions; **(b)** unit tests for feature engineering functions; **(c)** integration test for the full evening batch pipeline (data → features → model → ranking → risk → mock execution); **(d)** integration test for checkpoint pipeline; tests must pass before deployment to Hetzner and before any production code change |

**Phase 1 does NOT include:** real broker API credentials, real money transactions. OpenAlgo **is** deployed and integrated in Phase 1 — in mock/paper order mode. Phase 2 upgrades it to a live broker connection.

### Phase 2 — Broker Integration & Live Trading

| ID     | Deliverable |
|--------|-------------|
| P2-001 | Connect **OpenAlgo** (already self-hosted from Phase 1) to the selected live broker (Zerodha / Angel One / Upstox) — switch OpenAlgo from mock/paper mode to live broker credentials; no changes to the rest of the pipeline required |
| P2-002 | Implement HITL approval workflow via Telegram (Trial mode) |
| P2-003 | Deploy and run in Trial mode with ₹10,000 real funds |
| P2-004 | After Trial validation, promote to Production mode with ₹50,000 funds |
| P2-005 | Production monitoring and alerting |
| P2-006 | Upgrade the **intraday checkpoint process** to use broker API live position and price feeds (via OpenAlgo) in addition to Indian Stock Market API — enables real-time order fill confirmation and portfolio reconciliation |
| P2-007 | **Broker TOTP auto-login** (X-008) — automated daily session refresh using encrypted TOTP secret before market open |
| P2-008 | **Portfolio reconciliation:** at the start of each trading day (before 9:10 AM pre-market preparation) and at the end of each evening batch, compare the system's internal portfolio state (holdings, quantities, cash) against the broker-reported positions via OpenAlgo; **(a)** if discrepancies are detected (e.g., system shows a fill but broker does not, or broker shows a position the system doesn't track), log the mismatch with full details, send a **critical** Telegram alert, and **halt all new buy orders** until the mismatch is resolved; **(b)** mismatches in cash balance below **₹100** are treated as rounding and auto-reconciled; **(c)** the system's internal state is **never auto-corrected** from broker data without logging — every correction is auditable; **(d) Corporate action auto-adjustment:** before flagging a discrepancy, check if the mismatch is consistent with a stock split or bonus issue — detected by comparing the adjusted close price series for a discontinuity (e.g., price halved + share count doubled = 2:1 split); if a corporate action ratio is identified, **auto-adjust** the internal position (share count × ratio, entry price ÷ ratio), log the adjustment with the inferred ratio, and send a Telegram notification (not alert); no trading halt for corporate action adjustments; the trailing stop-loss floor (G-006) is also adjusted by the same ratio |

---

## 9. Technology Stack

| Layer | Technology | License / Cost |
|-------|-----------|---------------|
| ML Data Pipeline | Microsoft Qlib | Open-source (MIT), $0 |
| Prediction Model | LightGBM | Open-source (MIT), $0 |
| Portfolio Optimization | Qlib (Mean-Variance Optimizer) | Included in Qlib, $0 |
| Feature Engineering | pandas, numpy, TA-Lib | Open-source, $0 |
| Risk Engine | Custom Python rules | In-house, $0 |
| Trade Execution | OpenAlgo (self-hosted) | Open-source, $0 |
| Broker Connectivity | OpenAlgo adapters (Zerodha / Angel One / Upstox) | Free broker APIs |
| Market Data | BSE Bhavcopy, Indian Stock Market API (GitHub) | Free; real-time BSE/NSE data, no API key required |
| Scheduling | cron (Linux) | Built-in, $0 |
| Notifications | Telegram Bot API | Free |
| Runtime | Python 3.10+ | Open-source, $0 |

> **Key advantage:** The entire stack is open-source. No paid APIs. No recurring AI costs. All budget goes to infrastructure.

---

## 10. Infrastructure Requirements

| ID     | Requirement |
|--------|-------------|
| I-001  | **Hetzner CPX32** — AMD x86 EPYC, 4 vCPU, 8 GB RAM, 160 GB NVMe SSD; always-on 24x7; ~$12/month |
| I-001a | **VPS timezone:** system timezone must be set to **Asia/Kolkata (IST)** — all cron schedules, log timestamps, and trading calendar logic assume IST; verify with `timedatectl` after provisioning |
| I-001b | **Swap file:** configure a **4 GB swap file** on the NVMe SSD; this provides 12 GB total virtual memory (8 GB RAM + 4 GB swap); NVMe swap performance is acceptable for infrequent spikes; prevents the Linux OOM killer from silently terminating processes during Sunday retrain peaks (5–7 GB) or unexpected pandas/Qlib memory spikes; verify with `swapon --show` after provisioning |
| I-002  | x86 architecture required for binary compatibility with Qlib, TA-Lib, and LightGBM native extensions |
| I-003  | Python 3.10+ runtime environment |
| I-004  | Required libraries: Qlib, LightGBM, OpenAlgo SDK, pandas, numpy, TA-Lib |
| I-005  | Persistent storage for: market data history, model artifacts, portfolio state, transaction logs |
| I-006  | cron for daily scheduling |
| I-007  | Secure credential management for broker API keys and **TOTP secrets** (**Phase 2 onwards only** — Phase 1 uses OpenAlgo mock mode; no real broker credentials are required or stored in Phase 1); TOTP secret stored **encrypted at rest** using a machine-specific key; API keys and secrets must never appear in logs |
| I-008  | Telegram Bot integration for reports (all modes) and HITL approvals (Trial mode) |
| I-009  | Initial development and testing on local machine before Hetzner deployment |
| I-010  | **Process supervisor:** all system processes (intraday checkpoint, evening batch, OpenAlgo) must be managed by **systemd** with `Restart=on-failure`; the system must auto-start on VPS reboot (Hetzner maintenance) without human intervention; a **systemd watchdog** monitors process health; if a managed process is unresponsive for >60 seconds, systemd kills and restarts it; this is critical for the "24x7" availability requirement |
| I-011  | **VPS hardening (Phase 1):** SSH key-only access (password auth disabled); UFW firewall allowing only SSH (22), Telegram Bot outbound (443), and OpenAlgo (localhost-only binding, not exposed to public); all credentials and API keys stored using a secrets manager or encrypted config file — never in plaintext config or environment variables visible in `/proc`; log scrubbing: broker tokens, TOTP secrets, and API keys must never appear in any log file |
| I-011a | **OS security auto-updates:** enable `unattended-upgrades` (Debian/Ubuntu) for **security patches only** (not full dist-upgrades); this ensures the kernel, OpenSSL, and system libraries receive critical CVE fixes without manual intervention; auto-reboot after kernel updates may be configured for non-trading hours (e.g., Sunday 3:00 AM) to avoid market-hours disruption |
| I-012  | **Log rotation and disk cleanup:** configure **logrotate** for all application logs (daily rotation, 30-day retention, compress after 2 days); model artifacts: retain last 4 weekly models, archive older; feature matrices: retain current + previous only; disk usage alert via Telegram if NVMe usage exceeds 80% |
| I-013  | **Backup and disaster recovery:** **(a)** configure **daily automated backups** of critical state files: portfolio state, transaction/audit log, guardrail configuration, top-N membership history, and the latest model artifact; backup destination: a low-cost object storage (e.g., Hetzner Storage Box, ~€3/month for 100 GB) or an automated Hetzner snapshot; **(b)** backups run after the evening batch completes (~6:15 PM IST); **(c)** retain at least **30 days** of daily backups; **(d)** a monthly restore test (manual) is recommended to verify backup integrity; raw OHLCV data is NOT backed up — it can be re-bootstrapped from BSE Bhavcopy archives |

### 10.1 Memory Budget (8 GB VPS)

| Component | Estimated RAM |
|-----------|--------------|
| OS + system processes | ~1.0 GB |
| Qlib data pipeline (50 stocks, 2 yr history) | ~1.0–1.5 GB |
| LightGBM **inference** — evening batch only (not concurrent with checkpoints) | ~0.2–0.5 GB |
| LightGBM **full training** (Sunday retrain job only — non-trading day, no concurrent workloads) | ~1.0–2.0 GB peak |
| Feature matrix + predictions | ~0.5 GB |
| OpenAlgo + risk manager | ~0.5 GB |
| Intraday **Checkpoint 1** (9:30 AM) — includes LightGBM inference (market hours; not concurrent with evening batch) | ~0.5–0.8 GB |
| Intraday **Checkpoints 2 & 3** (11:00 AM / 2:00 PM) — price + VIX fetch + P&L rule checks only; no inference | ~0.1–0.2 GB |
| Headroom / buffer | ~1.7–3.0 GB |
| **Weekday evening batch peak** | **~4–5 GB** ✓ comfortably fits 8 GB |
| **Weekday CP1 checkpoint peak** (9:30 AM — full pipeline with inference) | **~4–5 GB** ✓ comfortably fits 8 GB |
| **Sunday retrain peak** | **~5–7 GB** ✓ fits 8 GB (no concurrent trading); 4 GB swap on NVMe provides safety net for unexpected spikes |

---

## 11. Error Handling & Recovery

| ID     | Requirement |
|--------|-------------|
| E-001  | If market data source is unavailable, skip that day’s pipeline and alert via Telegram; do **NOT** use stale data (>1 trading day old) for any trade decision; **impact on next-day checkpoints:** if the evening batch was skipped, the baseline plan on disk is flagged as stale (dated >1 trading day); at the following morning’s 9:10 AM pre-market preparation, detect the stale plan flag and **skip proposal generation at Checkpoint 1** — only stop-loss checks and drawdown monitoring run; a Telegram alert is sent noting that CP1 proposals are suppressed due to missing evening batch data |
| E-002  | If broker API / OpenAlgo is unreachable, queue pending trades to a **persistent on-disk queue** (JSON or SQLite) and retry with exponential backoff (initial: 30s, max: 5 minutes, max retries: 5); **on each retry, re-validate the limit price** against the latest available LTP — if the price has moved beyond the original limit buffer (0.3%), recalculate the limit price from the current LTP; if all retries fail, cancel the queued trades, log the failure, and send Telegram alert; on system crash during retry, the persistent queue survives and is replayed on restart |
| E-003  | If LightGBM model training fails, use last known good model; alert via Telegram |
| E-003a | **Model staleness guard:** track the timestamp of the last successful weekly retrain; if the model artifact is older than **14 calendar days** (i.e., 2 consecutive Sunday retrains have failed), send a **critical** Telegram alert; if older than **21 days**, halt all new buy proposals and send a Telegram alert — the system trades on a stale model at unacceptable risk; sell orders and stop-loss exits remain active |
| E-004  | If prediction produces anomalous outputs (extreme alpha scores), discard the batch and log |
| E-005  | System crash recovery: persist all state to disk so system can resume from last known state |
| E-006  | Send Telegram alert on any critical failure (data unavailable, model failure, broker down, crash) |
| E-007  | If the intraday checkpoint process crashes during market hours, log the failure, send a Telegram alert immediately with the current open position state, and attempt automatic restart in time for the next scheduled checkpoint; the daily batch pipeline is unaffected; **market-hours restart protocol:** on any startup (including systemd auto-restart or VPS reboot) that occurs during market hours (9:15 AM – 3:30 PM IST), the system must: **(a)** read the checkpoint log to determine which checkpoints have already completed today; **(b)** skip any missed checkpoint windows — do NOT attempt to "catch up" a missed CP; **(c)** run an **immediate stop-loss safety check** using the last known prices from the state file (not a full checkpoint — just G-006 stop-loss evaluation on all open positions); if any stop-loss floor is breached, submit market sell orders; **(d)** wait for the next scheduled checkpoint and resume the normal cycle; **(e)** send a Telegram alert summarising: restart time, checkpoints missed, open position state, and any stop-loss actions taken |
| E-008  | If the intraday checkpoint process cannot fetch live prices at a checkpoint, skip that checkpoint entirely and log the failure: **(a) Checkpoint 1** — proposal generation, inference, re-ranking, and all guard­rail checks are skipped for that window; **(b) Checkpoints 2 & 3** — stop-loss and drawdown monitoring are skipped for that window; in both cases the skipped monitoring is flagged in the daily report as an unmonitored window |
| E-009  | **Telegram fallback:** if Telegram API is unreachable for >5 minutes during a HITL approval window (Trial mode), apply the following fallback: **(a)** stop-loss exits auto-execute immediately (capital protection); **(b)** all other trade proposals are auto-cancelled (conservative default); **(c)** alerts are queued locally and retried every 5 minutes until Telegram is reachable; in Production/Sandbox mode (no HITL), Telegram unavailability does not affect trading — alerts are queued and sent when connectivity resumes |
| E-010  | **Graceful shutdown:** on SIGTERM or SIGINT, the system must: **(a)** complete any in-flight order submission (wait up to 30 seconds); **(b)** persist current portfolio state and pending order status to disk; **(c)** log the shutdown event with timestamp and open position summary; **(d)** exit cleanly; no new orders are initiated after receiving the signal |
| E-011  | **Startup self-diagnostic (pre-flight checks):** on every system start (including after crash recovery), verify before entering normal operation: **(a)** model artifact file exists and is valid (not corrupt / zero-byte); **(b)** portfolio state file exists and is parseable; **(c)** OpenAlgo is reachable (mock or live); **(d)** Telegram Bot API is reachable; **(e)** disk space > 10% free; **(f)** system clock is synchronized (NTP check — critical for IST cron timing); if any check fails, log the failure, send Telegram alert (if reachable), and halt startup until the issue is resolved |
| E-012  | **Checkpoint timeout:** each intraday checkpoint must complete within a configurable timeout (default: **10 minutes**); if LightGBM inference, API calls, or any step exceeds this timeout, abort the checkpoint, log the timeout, send Telegram alert, and allow the next scheduled checkpoint to proceed normally |

---

## 12. Regulatory & Compliance Notes

| ID     | Note |
|--------|------|
| C-001  | Automated trading via broker APIs is permitted by SEBI for retail investors |
| C-002  | Broker must be SEBI-registered and support free API-based order placement |
| C-003  | System must respect broker-imposed order throttling and rate limits |
| C-004  | Full audit trail of all decisions and transactions must be maintained |

---

## 13. Assumptions & Constraints

1. The system trades equities on BSE India only; NSE is out of scope for this version.
2. Maximum watchlist size: 50 stocks at any time.
3. Infrastructure costs (Hetzner CPX32, ~$12/month) are managed manually by the owner; no automated budget tracking is built into the system.
4. All data sources are free: BSE Bhavcopy, Indian Stock Market API (GitHub), broker-provided data.
5. Market data sources are accessible from Hetzner (no geo-blocking issues).
6. Tax calculations are estimates — actual tax filing remains the user's responsibility.
7. Mode promotion (Sandbox → Trial → Production) is a manual human decision.
8. The **system process is always-on 24x7** on the VPS (managed by systemd); trading logic activates only on trading days (Mon–Fri, excluding BSE holidays); on non-trading days the process remains running but idle (no data fetch, no inference, no trading).
9. Indian Stock Market API (GitHub) provides real-time BSE/NSE data at no cost and replaces all delayed-data sources for both the daily batch pipeline and the intraday checkpoint process. The specific GitHub repository must be confirmed and pinned to a stable version before development begins.
10. The stock watchlist starts with a manually configured seed list.
11. LightGBM model training for ≤50 stocks is computationally feasible on an 8 GB VPS.
12. OpenAlgo is self-hosted on the same VPS alongside the ML pipeline.
13. The system does not use any LLM or paid AI APIs — all intelligence is from Qlib + LightGBM.
14. A suitable free-API broker (via OpenAlgo) will be selected before Phase 2 begins.
15. The system uses **adjusted close prices** (available via Indian Stock Market API and BSE Bhavcopy) to handle corporate actions (stock splits, dividends, bonus issues) — no separate corporate action feed is required.
16. **Data retention:** A rolling minimum of 2 years of OHLCV data is maintained on disk at all times; data older than 5 years may be archived or deleted. Estimated storage: ~2–3 GB for 50 stocks × 5 years of daily data, well within the 160 GB NVMe.
17. **BSE trading calendar:** An annual holiday list is sourced manually (BSE publishes the list each December) and loaded into the system at the start of each calendar year.
18. The CPX32 AMD x86 EPYC instance is selected for x86 binary compatibility with Qlib, TA-Lib, and LightGBM native libraries; infrastructure costs are managed manually outside this system.
19. In Trial mode, the minimum viable order size is floor-capped at ₹500 regardless of the percentage-based G-003 calculation; trade proposals below this floor are skipped and logged. This is necessary because Indian broker per-order fees (~₹20 flat) represent ~10% of a ₹200 order, making sub-₹500 trades economically unviable.
20. The three intraday checkpoints operate in two tiers: Checkpoint 1 (9:30 AM) is the sole full-decision window (inference + re-ranking + proposals); Checkpoints 2 & 3 (11 AM, 2 PM) are lightweight P&L monitors (stop-loss, drawdown, VIX checks only). This design avoids running ML inference on structurally incomplete 90-minute-old intraday bars while retaining full mid-session risk protection.
21. **Indian Stock Market API (GitHub) is the sole intraday price source.** BSE Bhavcopy provides EOD data only. If this library becomes unmaintained, blocked by BSE, or stops working, intraday checkpoints cannot fetch live prices and will be skipped per E-008. Identifying a fallback intraday data source is deferred to a future phase. This is an accepted single-source dependency for the current design.
22. **SEBI retail algo exemption (<10 orders/second):** under SEBI circular SEBI/HO/MIRSD/MIRSD-PoD/P/CIR/2025/0000013, personal retail algorithmic systems operating below 10 orders per second are exempt from formal exchange-level strategy registration and approval. This system is naturally compliant — G-010 limits new proposals to a maximum of 10 per day, all submitted at a single checkpoint; actual order throughput never approaches 1 order/second, let alone 10. This exemption eliminates the requirement for exchange-level backtest audits or strategy code submission.
23. **Broker IP whitelisting:** under current SEBI network security mandates, brokers must whitelist the static IP address of any automated trading system. The Hetzner CPX32 VPS has a static public IP, satisfying this requirement. Two practical implications: **(a)** the VPS IP must be registered with the selected broker before Phase 2 (live trading) begins; **(b)** any future VPS migration (e.g., Hetzner region change, provider switch) requires a broker IP update request — brokers may restrict IP changes to once per calendar week; plan migrations for non-trading periods.

---

## 14. Design Considerations

The following items are **not requirements** — they are known decisions and trade-offs to be resolved during system design and implementation. They are recorded here so they are not lost between requirements and design phases.

| # | Item | Context |
|---|------|---------|
| DC-1 | **Share quantity rounding** | BSE allows whole shares only. For high-priced stocks (e.g., ₹2,000/share), G-003 (max ₹1,500/order at ₹50K capital) cannot buy even 1 share. Implementation must round down and skip if <1 share is affordable. |
| DC-2 | **T+1 settlement** | India uses T+1 equity settlement. After Tier 2 liquidation, sale proceeds settle next business day. During the 5-day cooling-off (S-010) this is not a problem. Edge cases for same-stock rebuy are already prevented by M-008 (3-day holding period). |
| DC-3 | **State persistence format** | "Persist to disk" appears throughout the document without format commitment. Design must choose between JSON files, SQLite, or structured binary. Key trade-offs: JSON is human-readable and debuggable; SQLite supports queries and atomicity; both are lightweight. |
| DC-4 | **Deployment strategy** | How new code reaches the VPS (git pull + restart, rsync, Docker container) is a design decision. Must ensure zero-downtime deployment during non-trading hours. |
| DC-5 | **API rate limiting** | Indian Stock Market API says "no API key required" but may throttle requests. Empirical testing needed during development to determine actual rate limits and appropriate request spacing. |
| DC-6 | **Backtest slippage modelling** | P1-017 specifies a configurable slippage estimate (default 0.1%). During implementation, model slippage as a **randomized range** (e.g., 0.05%–0.2%) rather than a fixed constant — prevents overfitting to perfect fills. |
| DC-7 | **Feature stability on partial bars** | CP1 runs LightGBM inference on a partial intraday bar (~30 minutes of data at 9:30 AM). Features like RSI, MACD, and moving averages may produce edge-case values on incomplete bars. Validate feature stability during backtesting and consider using only features that are meaningful on partial-day data at checkpoint time. |

---

## 15. Glossary

| Term | Definition |
|------|------------|
| Alpha Score | A model's predicted excess return for a stock |
| Backtesting | Running the complete trading strategy against historical data to validate profitability before live deployment |
| BSE | Bombay Stock Exchange |
| Drawdown | Peak-to-trough decline in portfolio value |
| Feature Engineering | Transforming raw price data into statistical indicators for ML input |
| Guardrail | A hard-coded risk limit that cannot be overridden by the model |
| HITL | Human-In-The-Loop — human approval required before action |
| Indian Stock Market API | Open-source GitHub library providing real-time BSE/NSE stock and index data at no cost; specific repository to be confirmed before development begins |
| India VIX | India Volatility Index — measures expected near-term market volatility |
| Intraday Trading Checkpoints | Three intraday windows per trading day (9:30 AM, 11:00 AM, 2:00 PM) operating in two tiers: Checkpoint 1 (9:30 AM) is the full decision window (inference, re-ranking, proposals, execution); Checkpoints 2 & 3 are P&L monitoring windows (stop-loss, drawdown, and VIX regime checks only) |
| LightGBM | Light Gradient Boosting Machine — a fast, accurate ML algorithm for tabular data |
| Mean-Variance Optimization | Portfolio allocation method that balances expected return vs risk |
| OHLCV | Open, High, Low, Close, Volume — standard price bar data |
| OpenAlgo | Open-source broker-agnostic trade execution platform for Indian markets |
| Production | Full autonomous trading mode with allocated real funds |
| Pseudo Transaction | A mock/paper order submitted through OpenAlgo in Sandbox mode — processed by OpenAlgo but not forwarded to any real broker; fill is simulated at real-time price |
| Qlib | Microsoft's open-source quantitative investment research platform |
| RSI | Relative Strength Index — a momentum oscillator |
| Sandbox | Simulation mode with no real money |
| TOTP | Time-based One-Time Password — used for daily broker API authentication |
| Trial | Limited real-money mode with HITL oversight |
| Watchlist | Curated list of stocks (≤50) actively monitored and traded by the system |

---

*End of Requirements Document — v3.7*
