# Requirements for Auto-Trader (AT) — Synthesized Reference

> **Source**: Requirements_Formal.md v3.7 (638 lines) + 12 verification answers.
> For full traceability, see [requirement-verification-questions.md](requirement-verification-questions.md).

---

## 1. Intent Analysis Summary

| Field | Value |
|---|---|
| **Request** | Build a fully automated quantitative trading system for BSE India |
| **Type** | New Project (Greenfield) |
| **Scope** | System-wide — 7 major components, 2 delivery phases, 3 operating modes |
| **Complexity** | Complex — ML pipeline, real-time checkpoints, broker integration, risk engine, Telegram HITL |

---

## 2. Clarification Decisions (from Verification Q&A)

These answers override or supplement the formal requirements:

| # | Topic | Decision | Impact |
|---|---|---|---|
| Q1 | Data APIs | **AlphaVantage** for research/historical; **Kotak Neo API** for real-time prices and trading | Replaces "Indian Stock Market API (GitHub)" throughout. Affects D-002, IM-003, Assumption 9/21 |
| Q2 | Seed watchlist | BSE 100 filtered by X-009 liquidity threshold (≥₹50 lakh/day avg) | ~30-40 stocks initial |
| Q3 | State persistence | **SQLite** for all state: portfolio, plans, audit trails, checkpoint logs | Resolves DC-3 |
| Q4 | Deployment | **Git pull + systemd restart** over SSH | Resolves DC-4 |
| Q5 | Candidate pool | BSE 200/500 filtered by X-009 liquidity threshold | 50-150 stocks for Saturday screener |
| Q6 | Broker | **Kotak Neo** (Phase 2) | Replaces generic OpenAlgo broker selection |
| Q7 | Telegram bot | Create new bot as part of Phase 1 | Setup instructions needed |
| Q8 | Project structure | AI to select best practice for production Python on VPS | Decision at design time |
| Q9 | Backtest thresholds | AI to recommend based on BSE market characteristics | Replaces default Sharpe ≥ 0.3 / DD < 25% |
| Q10 | Feature set | AI to design optimal feature set for LightGBM + BSE swing trading | Replaces default 7-feature list |
| Q11 | OpenAlgo paper mode | Research and confirm during design phase | May need internal mock execution layer |
| Q12 | Dev environment | **Windows native** (PowerShell/CMD) | No WSL2 |

---

## 3. System Overview

An automated quantitative trading system that runs a deterministic closed-loop pipeline daily on a Hetzner CPX32 VPS (4 vCPU, 8 GB RAM, 160 GB NVMe):

1. **Observe** — Collect BSE market data (AlphaVantage + Kotak Neo API + BSE Bhavcopy)
2. **Learn** — Train/retrain LightGBM model (weekly Sunday retrain)
3. **Decide** — Rank stocks by predicted alpha scores, optimize portfolio via Qlib MVO
4. **Protect** — Filter through hard-coded risk guardrails (G-001 through G-013)
5. **Act** — Execute via OpenAlgo (mock in Phase 1, Kotak Neo live in Phase 2)

Two distinct flows:
- **Evening Batch** (post-close, trading days): Data → Features → Prediction → Ranking → MVO → Risk → Baseline Plan → Telegram Report
- **Intraday Checkpoints** (market hours): CP1 (9:30 AM full decision) + 5 hourly monitoring (Mon-1 through Mon-5, 10:30 AM – 2:30 PM, P&L monitoring only)

---

## 4. Functional Requirements

### 4.1 Data Ingestion (D-001 through D-009)
- Daily OHLCV for active watchlist (≤50 stocks) from BSE Bhavcopy (EOD) and Kotak Neo API (intraday)
- AlphaVantage for historical research data
- Sensex index data for relative strength benchmark and volatility regime computation
- BSE trading calendar (annual holiday list for scheduler)
- Intraday data validation: reject stale/zero/extreme prices (D-005a)
- Historical data bootstrap: 2-year OHLCV bulk download before first run (Q-006)

### 4.2 Data Preparation — Qlib (Q-001 through Q-006)
- Microsoft Qlib for structured dataset management
- Align by trading date, handle holidays/missing data
- Rolling training windows; minimum 2-year history
- Structured storage for feature engineering input

### 4.3 Feature Engineering (F-001 through F-005)
- Technical features: momentum, volatility, RSI, MACD, volume spikes, moving averages, relative strength vs Sensex
- AI to design optimal feature set for LightGBM + BSE swing trading (per Q10 decision)
- Feature matrix: stocks × features × dates
- Deterministic computation, configurable and extensible

### 4.4 Model Training & Prediction — LightGBM (M-001 through M-008)
- Predict expected return (alpha score) per stock, configurable N-day horizon (default: 5 days)
- Weekly full retrain on Sundays; daily inference uses persisted model artifact
- Must complete within minutes on 8 GB VPS
- Persist model artifacts for crash recovery
- Log performance metrics (prediction accuracy, Sharpe ratio)
- Minimum 3-day holding period (M-008) before ranking-based sells
- Anomalous output detection and discard

### 4.5 Stock Ranking & Portfolio Optimization (S-001 through S-010)
- Rank by alpha score; select top-N candidates (default: 5, must be ≤ G-002)
- Qlib Mean-Variance Optimization for position weights
- Position exit: trailing stop-loss (G-006) + ranking-based exit at CP1
- Cold-start ramp: 5-day capital deployment (20%/40%/60%/80%/100%)
- G-005 Tier 2 recovery: 5-day cooling-off → peak reset → ramp restart

### 4.6 Risk Manager — Guardrails (G-001 through G-013)
- **Position sizing**: Max 5% per stock (G-001), max 10 positions (G-002), max 3% per order (G-003)
- **Drawdown**: 3% daily loss halt (G-004), 10% Tier 1 buy-halt with hysteresis (G-005a), 15% Tier 2 full liquidation (G-005b)
- **Stop-loss**: Trailing 5% below highest-seen price (G-006)
- **Volatility regime** (Sensex realized vol): >30% annualized halve positions (G-007), >45% halt buys (G-008)
- **Circuit breaker**: Sensex ≥10% intraday drop → halt (G-009)
- **Operational**: Max 10 proposals/day Production, 3 Trial (G-010); max 50 watchlist (G-011)
- All configurable per mode (Sandbox/Trial/Production) with change logging (G-012)
- Telegram alerts on every trigger (G-013)

### 4.7 Trade Execution — OpenAlgo (X-001 through X-009)
- Broker-agnostic via OpenAlgo; **Kotak Neo** for Phase 2
- Sandbox: OpenAlgo mock/paper mode (verify support — Q11)
- Limit orders at LTP ± 0.3%; stop-loss exits as market orders
- CP1-only proposals; unfilled limit orders auto-cancelled at Mon-1
- Broker session management: 9:00 AM TOTP refresh (Phase 2+)
- Liquidity validation: BSE-specific avg daily value ≥ ₹50 lakh (X-009)

### 4.8 Reporting Pipeline (RP-001 through RP-008)
- Daily portfolio snapshot, P&L, model accuracy, decision traceability
- Full audit trail with model version, features, guardrail pass/fail per trade
- Telegram daily summary (trading days, post-close)
- Daily heartbeat: 9:15 AM trading days, 12:00 PM weekends/holidays
- Persist daily top-N membership list (RP-008) for Saturday screener

### 4.9 Intraday Trading Checkpoints (IM-001 through IM-013)
- **CP1 (9:30 AM)**: Full pipeline — fetch prices → compute volatility regime → LightGBM inference → re-rank → proposals → guardrails → execute/HITL
- **Hourly Monitoring CPs (10:30 AM, 11:30 AM, 12:30 PM, 1:30 PM, 2:30 PM)**: P&L monitoring — prices → unfilled order cleanup (first monitoring CP only) → stop-loss → drawdown → volatility regime. No new buy proposals. Max gap between checkpoints: 1 hour.
- Pre-market preparation at 9:10 AM (volatility regime check, regime filter on baseline plan)
- 10-minute timeout per checkpoint (E-012)
- Full logging per checkpoint cycle (IM-013)

### 4.10 Watchlist Management (W-001 through W-009)
- Max 50 stocks at any time
- Seed: BSE 100 filtered by liquidity (per Q2)
- Saturday screener (independent cron): candidate pool additions (volume check) + removal candidates (absent from top-N for 4+ weeks)
- Candidate pool: BSE 200/500 filtered by liquidity (per Q5)
- Trial/Sandbox: Telegram approval; Production: auto-apply with notification
- Auto-removal for suspended/delisted/circuit-halted stocks >3 days (W-007) with frozen holding logic
- New stock addition triggers 2-year historical data download (W-009)

---

## 5. Operating Modes

| Attribute | Sandbox | Trial | Production |
|---|---|---|---|
| **Capital** | ₹50,000 (simulated) | ₹10,000 (real) | ₹50,000 (real) |
| **Orders** | OpenAlgo mock | Real via Kotak Neo | Real via Kotak Neo |
| **HITL** | None | Telegram approval (15 min timeout) | None |
| **Duration** | Min 1 month | Until satisfied | Ongoing |
| **Watchlist changes** | Telegram approval | Telegram approval | Auto-apply + notification |
| **Min order floor** | — | ₹500 | — |
| **Exit criteria** | Manual promotion | Manual promotion | — |

Advisory Sandbox exit benchmarks: ≥22 trading days, annualised Sharpe ≥ 0.5, max DD < 10%, no guardrail violations.

---

## 6. Delivery Phases

### Phase 1 — Sandbox System (21 deliverables: P1-001 through P1-021)
- Full pipeline: data ingestion → Qlib → features → LightGBM → ranking → MVO → risk → OpenAlgo mock → reports
- Intraday checkpoints (3 daily, 2-tier)
- Watchlist management + Saturday screener
- Backtesting framework (P1-017) with transaction cost modelling
- systemd services, VPS hardening, startup diagnostics, graceful shutdown
- Testing suite: unit tests (guardrails, features), integration tests (evening batch, checkpoints)
- Local dev (Windows native) → Hetzner deployment
- **New Telegram bot** created as part of this phase (per Q7)

### Phase 2 — Broker Integration & Live Trading (P2-001 through P2-008)
- Connect OpenAlgo to **Kotak Neo** (swap mock → live credentials)
- HITL Telegram approval workflow (Trial mode)
- Trial with ₹10,000 → Production with ₹50,000
- Broker TOTP auto-login (X-008)
- Portfolio reconciliation (P2-008) with corporate action auto-adjustment

---

## 7. Non-Functional Requirements

### Performance
- Evening batch + 3 intraday checkpoints within 8 GB RAM budget (see memory table in formal spec §10.1)
- LightGBM inference in milliseconds; full checkpoint ≤ 10 min timeout
- Sunday retrain: 5-7 GB peak, fits with 4 GB swap

### Reliability & Recovery
- systemd `Restart=on-failure` with watchdog (I-010)
- Full crash recovery from persisted state (E-005)
- Market-hours restart protocol (E-007): detect completed checkpoints, skip missed, immediate stop-loss safety check
- Broker API retry with exponential backoff (E-002)
- Model staleness guard: alert at 14 days, halt buys at 21 days (E-003a)
- Telegram fallback (E-009): auto-execute stop-losses, auto-cancel other proposals

### Security
- SSH key-only access, UFW firewall, OpenAlgo localhost-only (I-011)
- Encrypted credential storage (never plaintext, never in logs)
- OS security auto-updates (I-011a)
- Log scrubbing for tokens/secrets

### Maintainability
- Log rotation (30-day retention, compress after 2 days) (I-012)
- Model artifact retention: last 4 weekly models
- Daily automated backups of critical state to Hetzner Storage Box (I-013)

### Persistence — SQLite (per Q3)
- Portfolio state, transaction logs, audit trails, checkpoint logs
- Baseline plans, guardrail config history, top-N membership history
- Configuration files may remain human-readable (JSON/YAML)

### Observability — Per-Module Logging
- System produces **separate log files per module** (minimum 3-4): data pipeline, model/feature engine, execution/risk engine, scheduler/orchestrator
- Each log file contains both error-level and debug-level entries with timestamps
- Log levels configurable per module (DEBUG, INFO, WARNING, ERROR, CRITICAL)
- Log rotation via logrotate: daily rotation, 30-day retention, compress after 2 days
- Structured log format enabling quick triage: timestamp, module, level, message
- Admin can tail any module log independently to diagnose issues without wading through unrelated output

### Deployment — Git pull + systemd (per Q4)
- VPS has git clone; deploy via SSH → git pull → systemd restart
- Zero-downtime during non-trading hours

---

## 8. Technology Stack

| Layer | Technology |
|---|---|
| ML Data Pipeline | Microsoft Qlib (MIT) |
| Prediction Model | LightGBM (MIT) |
| Portfolio Optimization | Qlib Mean-Variance Optimizer |
| Feature Engineering | pandas, numpy, TA-Lib |
| Risk Engine | Custom Python rules engine |
| Trade Execution | OpenAlgo (self-hosted) → **Kotak Neo** |
| Market Data | AlphaVantage (research) + Kotak Neo API (live) + BSE Bhavcopy (EOD) |
| Scheduling | cron (Linux) |
| Notifications | Telegram Bot API (new bot — Phase 1) |
| State Persistence | **SQLite** |
| Runtime | Python 3.10+ |
| Dev Environment | Windows native (PowerShell/CMD) |
| Production Host | Hetzner CPX32 (4 vCPU, 8 GB, 160 GB NVMe, IST timezone) |

---

## 9. Open Design Decisions (to resolve during Application Design)

| # | Decision | Context | Resolution Status |
|---|---|---|---|
| DC-1 | Share quantity rounding | BSE whole shares only; G-003 may yield <1 share for high-priced stocks | Defer to Functional Design (Unit 2) |
| DC-2 | T+1 settlement edge cases | India T+1; covered by M-008 3-day hold but verify edge cases | Defer to Functional Design (Unit 2) |
| DC-5 | API rate limiting | AlphaVantage and Kotak Neo rate limits — empirical testing needed | Defer to Functional Design (Unit 1) |
| DC-6 | Backtest slippage modelling | Randomized range (0.05%-0.2%) vs fixed 0.1% constant | Resolved: BacktestEngine supports configurable slippage |
| DC-7 | Feature stability on partial bars | CP1 runs on ~30 min of data; validate feature meaningfulness | Resolved: FeatureEngine.validate_feature_stability() |
| Q8 | Python project structure | AI to recommend at design time | Resolved: unit-of-work.md code organization |
| Q9 | Backtest thresholds | AI to recommend based on BSE characteristics | Defer to Functional Design (Unit 2) |
| Q10 | Optimal feature set | AI to design for LightGBM + BSE swing trading | Defer to Functional Design (Unit 2) |
| Q11 | OpenAlgo paper mode | Research whether supported; fallback: internal mock layer | Defer to Functional Design (Unit 3) |
