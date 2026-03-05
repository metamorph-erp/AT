# User Stories for Auto-Trader (AT)

> **Scope**: Phase 1 (MVP — Sandbox System). Phase 2 stories are listed as a future epic.
> **Format**: High-level epics with sub-stories. Acceptance criteria as informal bullet points.
> **Persona mapping**: Only on stories with direct user interaction.

---

## Epic 1: Data Pipeline

### US-1.1: Historical Data Bootstrap
As a trader, I want the system to bulk-download 2 years of OHLCV history for all seed watchlist stocks so that the ML model has sufficient training data before the first run.

**Acceptance Criteria:**
- Downloads >=2 years of daily OHLCV from AlphaVantage (`TIME_SERIES_DAILY`, `outputsize=full`) for all seed watchlist stocks
- Data is loaded into the Qlib dataset structure
- Validates data completeness (no gaps >3 consecutive trading days without flagging)
- Logs download progress and any failed symbols
- System blocks Sandbox mode entry until bootstrap is confirmed complete

### US-1.2: Daily EOD Data Collection
As a system, I want to collect end-of-day OHLCV data for all watchlist stocks and Sensex index after market close so that the evening batch pipeline has fresh data.

**Acceptance Criteria:**
- Runs daily at 4:00 PM IST on trading days only (per BSE calendar)
- Fetches OHLCV for ≤50 watchlist stocks from BSE Bhavcopy
- Fetches Sensex closing level (also used for volatility regime computation)
- Validates data integrity (D-005): rejects corrupt, incomplete, zero, or negative records
- Stores raw data in local persistent storage
- Skips non-trading days (weekends, BSE holidays)

### US-1.3: Intraday Price Fetch
As a system, I want to fetch live intraday prices at each checkpoint so that the risk engine and (at CP1) the model can use current market data.

**Acceptance Criteria:**
- Phase 1: fetches partial intraday OHLCV bars via AlphaVantage for held symbols plus today's baseline proposal symbols
- Phase 2: fetches partial intraday OHLCV bars via Kotak Neo API
- Fetches current Sensex level (for volatility regime and circuit breaker checks)
- Applies D-005a validation: rejects stale, zero, negative, or >20% deviation prices
- If >30% of stocks fail validation, skips the entire checkpoint (per E-008)
- Runs at 9:30 AM (CP1), then hourly monitoring checkpoints at 10:30 AM, 11:30 AM, 12:30 PM, 1:30 PM, and 2:30 PM IST

### US-1.4: BSE Trading Calendar
As a system, I want to maintain a BSE trading calendar so that the scheduler correctly skips non-trading days.

**Acceptance Criteria:**
- Annual holiday list loaded at system start
- Scheduler uses calendar to skip weekends and BSE holidays
- Calendar is refreshable at the start of each calendar year
- All cron schedules respect the calendar

---

## Epic 2: Feature Engineering & Model

### US-2.1: Feature Engineering Pipeline
As a system, I want to compute technical features from raw price data so that LightGBM has meaningful input signals.

**Acceptance Criteria:**
- Produces features including: momentum, volatility, RSI, MACD, volume spikes, moving averages, relative strength vs Sensex
- AI-designed optimal feature set for BSE swing trading (per Q10 decision)
- Outputs a feature matrix (stocks × features × dates)
- All computation is deterministic — no randomness or external API calls
- Feature set is configurable and extensible without changing core pipeline
- Handles partial bars at CP1 with validated feature stability (DC-7)

### US-2.2: LightGBM Training
As a system, I want to train the LightGBM model weekly so that predictions stay current with evolving market patterns.

**Acceptance Criteria:**
- Weekly full retrain runs on Sunday at 2:00 AM IST
- Uses rolling 2-year training window
- Completes within minutes on 8 GB VPS (+ 4 GB swap)
- Persists model artifact to disk for crash recovery and versioning
- Logs performance metrics (prediction accuracy, Sharpe ratio)
- Retains last 4 weekly model artifacts (older archived)
- On failure, falls back to last known good model + Telegram alert (E-003)

### US-2.3: LightGBM Prediction (Daily + CP1)
As a system, I want to run LightGBM inference to produce alpha scores for all watchlist stocks so that the ranking and optimizer can generate trade proposals.

**Acceptance Criteria:**
- Evening batch: runs inference using current model artifact on full EOD data
- CP1 (9:30 AM): runs inference on partial intraday bar (incremental feature update, not full reload)
- Produces alpha score (expected N-day return) per stock
- Discards batch and logs if anomalous outputs detected (M-007)
- Model staleness guard: alert at 14 days, halt buys at 21 days without successful retrain (E-003a)

---

## Epic 3: Ranking, Optimization & Execution

### US-3.1: Stock Ranking & Portfolio Optimization
As a system, I want to rank stocks by alpha score and compute optimal position weights so that the system knows what and how much to trade.

**Acceptance Criteria:**
- Ranks all watchlist stocks by predicted alpha score
- Selects top-N candidates (default: 5, must be ≤ G-002)
- Uses Qlib Mean-Variance Optimization for position weights
- Considers individual stock volatility, cross-stock correlations, total portfolio risk
- Outputs trade actions (buy/sell) with specific share quantities
- Applies S-009 cold-start ramp when starting with zero holdings
- Re-held stocks in top-N produce no new buy order (S-008)

### US-3.2: Trade Execution via OpenAlgo (Mock Mode)
As a system, I want to submit trade proposals through OpenAlgo in mock/paper mode so that Sandbox simulation exercises the full execution path without real money.

**Acceptance Criteria:**
- Submit limit orders to OpenAlgo at LTP ± 0.3% buffer (buys LTP+, sells LTP−)
- Stop-loss exits submitted as market orders
- OpenAlgo processes orders in mock mode — no real broker connection
- Fills simulated at fetched real-time price
- Logs every order: status (filled/rejected/partial), timestamp, price, quantity
- Validates stock is held before submitting sell orders (X-006)
- Unfilled limit orders from CP1 auto-cancelled at Mon-1 (IM-006a)
- Liquidity validation before buy orders: BSE avg daily value ≥ ₹50 lakh (X-009)

### US-3.3: Position Exit Logic
As a system, I want to apply stop-loss and ranking-based exit rules so that losing positions are cut and underperforming holdings are rotated.

**Acceptance Criteria:**
- Trailing stop-loss (G-006): floor at 5% below highest-seen price since entry, ratchets up, never down
- Highest-seen price uses session high-so-far from IM-003 (not LTP)
- Stop-loss checked at every checkpoint (CP1, Mon-1 through Mon-5)
- Ranking-based exit: if stock drops out of top-N at CP1, ranking-based sell proposed
- Minimum holding period (M-008): 3 trading days before ranking-based sell eligibility
- Stop-loss always takes precedence over ranking retention

---

## Epic 4: Risk Engine

### US-4.1: Position Sizing Guardrails
As a system, I want to enforce hard position sizing limits so that no single stock or order can over-concentrate the portfolio.

**Acceptance Criteria:**
- G-001: Max total holding per stock ≤ 5% of total capital
- G-002: Max 10 simultaneous open positions
- G-003: Max single order ≤ 3% of total capital
- All limits configurable per mode with change logging (G-012)
- Blocked proposals logged with full reasoning

### US-4.2: Drawdown Protection
As a system, I want to enforce daily loss limits and total drawdown protection so that the portfolio is protected from catastrophic losses.

**Acceptance Criteria:**
- G-004: Daily loss ≥3% → halt new buys for the day; sells/stop-losses remain active
- G-005 Tier 1 (10%): Halt new buys; resume when drawdown recovers below 8% (2pp hysteresis)
- G-005 Tier 2 (15%): Market sell all tradeable positions (frozen excluded); 5-day cooling-off; peak reset; cold-start ramp (S-010)
- Tier 2 overrides G-004
- Telegram alerts on every trigger

### US-4.3: Volatility Regime Filters
As a system, I want Sensex-based volatility regime filters so that the system reduces exposure in high-volatility markets.

**Acceptance Criteria:**
- G-007: Sensex 20-day annualized realized vol > 30% → halve all position sizes
- G-008: Sensex 20-day annualized realized vol > 45% → halt all new buys; allow sells/exits only
- G-009: Sensex ≥10% intraday drop → halt all activity; resume next day (Sandbox/Trial: Telegram confirmation; Production: auto)
- Volatility regime checked at pre-market (9:10 AM) and every checkpoint
- Regime recovery within same day does not restore sizes (conservative)

### US-4.4: Operational Limits & Alerting
As a system, I want to enforce operational limits and alert on every guardrail activation so that the trader has full visibility.

**Acceptance Criteria:**
- G-010: Max proposals/day: unlimited (Sandbox), 3 (Trial, all at CP1), 10 (Production, all at CP1)
- G-011: Max watchlist size: 50
- G-013: Telegram alert on every guardrail trigger with details

---

## Epic 5: Scheduling & Checkpoints

### US-5.1: Evening Batch Orchestration
As a system, I want the evening batch to run automatically post-close on trading days so that a fresh baseline plan is ready for the next day.

**Acceptance Criteria:**
- 4:00 PM: Data collection → 4:30 PM: Features + prediction → 5:00 PM: Ranking + MVO → 5:15 PM: Persist plan + state → 5:30 PM: Telegram daily report
- Runs only on trading days (BSE calendar)
- Persists baseline plan, top-N membership history (RP-008), and full system state
- If data unavailable, skip pipeline + Telegram alert (E-001); flags plan as stale

### US-5.2: Intraday Checkpoint Orchestration
As a system, I want one trade checkpoint and five hourly monitoring checkpoints per trading day so that the system executes trades at open and monitors risk with no more than a 1-hour gap.

**Acceptance Criteria:**
- 9:10 AM: Pre-market preparation (volatility regime check, regime filter on baseline plan — no orders)
- 9:30 AM (CP1): Full decision — inference, re-ranking, proposals, guardrails, execution
- 10:30 AM (Mon-1): P&L monitoring — unfilled order cleanup, stop-loss, drawdown, volatility regime
- 11:30 AM (Mon-2): P&L monitoring — stop-loss, drawdown, volatility regime
- 12:30 PM (Mon-3): P&L monitoring — stop-loss, drawdown, volatility regime
- 1:30 PM (Mon-4): P&L monitoring — stop-loss, drawdown, volatility regime
- 2:30 PM (Mon-5): P&L monitoring — stop-loss, drawdown, volatility regime
- 3:30 PM: Cancel remaining HITL-awaiting proposals; day ends
- Maximum gap between any two checkpoints: 1 hour
- Each checkpoint completes within 10-minute timeout (E-012)
- Independent from evening batch (IM-012)

### US-5.3: Weekly Jobs
As a system, I want the Saturday screener and Sunday retrain to run as independent cron jobs so that the watchlist stays fresh and the model stays current.

**Acceptance Criteria:**
- Saturday 10:00 AM: Evaluate candidate pool additions + removal candidates
- Saturday 10:30 AM: Send consolidated Telegram message (approval in Sandbox/Trial)
- Sunday 2:00 AM: LightGBM full retrain
- Both are independent cron jobs, no dependency on trading day processes

---

## Epic 6: Watchlist Management

### US-6.1: Seed Watchlist Initialization
**Persona: Trader/Owner**

As a trader, I want the system to initialise from the BSE 100 filtered by liquidity so that I start with a well-known, liquid set of stocks.

**Acceptance Criteria:**
- BSE 100 index constituents filtered by X-009 liquidity threshold (BSE avg daily value ≥ ₹50 lakh)
- Result is ~30-40 stocks
- Seed list stored as the initial active watchlist
- Historical data bootstrap (US-1.1) triggered for all seed stocks

### US-6.2: Saturday Screener
**Persona: Trader/Owner** (receives Telegram approval request)

As a trader, I want a weekly screener that proposes watchlist additions and removals so that the watchlist evolves without manual effort.

**Acceptance Criteria:**
- Additions: evaluates BSE 200/500 candidate pool for volume criterion
- Removals: flags stocks absent from top-N for ≥4 consecutive weeks (reading RP-008 history)
- Consolidated Telegram message with all candidates
- Sandbox/Trial: awaits human approval via Telegram (24-hour window, default: not applied)
- W-009: Approved additions trigger 2-year historical data download
- All changes logged with timestamp and reason

### US-6.3: Auto-Removal of Suspended/Delisted Stocks
As a system, I want automatic removal of stocks suspended, delisted, or circuit-breaker-halted for >3 days so that the watchlist stays tradeable.

**Acceptance Criteria:**
- Detects suspension/delisting/halt >3 consecutive trading days
- Auto-removes from watchlist in all modes
- Open positions become frozen holdings (excluded from re-ranking, not counted toward G-002)
- Stop-loss continues against last available price
- On resume: market sell to liquidate frozen position
- Telegram alerts on freeze, resume, and liquidation

---

## Epic 7: Reporting & Notifications

### US-7.1: Daily Telegram Report
**Persona: Trader/Owner**

As a trader, I want a daily Telegram summary after market close so that I have full visibility into the day's activities without checking logs.

**Acceptance Criteria:**
- Portfolio snapshot: holdings, quantities, market values
- Current P&L with estimated tax impact
- Model prediction accuracy and cumulative performance
- Trades executed, skipped, or blocked with reasoning
- Guardrail activations
- Sent post-close on trading days only (all modes)

### US-7.2: Daily Heartbeat
**Persona: Trader/Owner**

As a trader, I want heartbeat messages so that I know instantly if the system goes offline.

**Acceptance Criteria:**
- 9:15 AM IST on trading days: "AT alive — pre-market starting"
- 12:00 PM IST on weekends/holidays: "AT alive — idle"
- Brief (one line) to avoid alert fatigue
- Missing heartbeat = earliest signal of system unavailability

### US-7.3: Audit Trail & Decision Traceability
As a system, I want a complete audit trail so that any past trade decision can be fully reconstructed.

**Acceptance Criteria:**
- Every trade log entry records: model artifact version, input feature snapshot, guardrails evaluated (pass/fail), portfolio state at decision time (RP-005a)
- Full transaction log accessible at any time
- All guardrail config changes logged with timestamp, old/new value, mode affected (G-012)
- Stored in SQLite

---

## Epic 8: Error Handling & Recovery

### US-8.1: Crash Recovery & State Persistence
As a system, I want to persist all critical state to disk so that the system can resume from last known state after any crash.

**Acceptance Criteria:**
- Portfolio state, pending orders, baseline plan, checkpoint progress persisted to SQLite
- On startup: self-diagnostic (E-011) — model artifact, portfolio state, OpenAlgo reachability, Telegram reachability, disk space, NTP clock sync
- Market-hours restart: detect completed checkpoints, skip missed, immediate stop-loss safety check (E-007)
- Graceful shutdown: complete in-flight orders (30s), persist state, log shutdown (E-010)

### US-8.2: Data Source & API Failure Handling
As a system, I want robust handling of data source and API failures so that the system degrades gracefully without using stale data.

**Acceptance Criteria:**
- E-001: Data source unavailable → skip pipeline, Telegram alert, flag plan as stale
- E-002: OpenAlgo unreachable → persistent on-disk queue, exponential backoff (5 retries), re-validate limit price on each retry
- E-008: Price fetch failure at checkpoint → skip checkpoint entirely, log as unmonitored window
- E-009: Telegram unreachable during HITL → auto-execute stop-losses, auto-cancel other proposals, queue alerts for retry

### US-8.3: Checkpoint Timeout
As a system, I want a timeout on each checkpoint so that a hung process cannot block subsequent checkpoints.

**Acceptance Criteria:**
- 10-minute timeout per checkpoint (configurable)
- On timeout: abort checkpoint, log, Telegram alert, allow next checkpoint to proceed

---

## Epic 9: Infrastructure & Deployment (Phase 1)

### US-9.1: Local Development Environment
**Persona: System Administrator**

As an administrator, I want the system to develop and test locally on Windows native so that I can iterate quickly before deploying to the VPS.

**Acceptance Criteria:**
- Works on Windows native with PowerShell/CMD
- Python 3.10+ with all dependencies installable on Windows
- SQLite for persistence (cross-platform)
- Can run evening batch and checkpoints in test mode locally

### US-9.2: VPS Deployment & Hardening
**Persona: System Administrator**

As an administrator, I want to deploy to Hetzner CPX32 with proper hardening so that the system runs securely and reliably.

**Acceptance Criteria:**
- Hetzner CPX32: 4 vCPU, 8 GB RAM, 160 GB NVMe, IST timezone (I-001a)
- 4 GB swap file configured (I-001b)
- SSH key-only access, password auth disabled
- UFW firewall: SSH (22), Telegram outbound (443), OpenAlgo localhost-only
- Encrypted credential storage (never plaintext, never in logs/`/proc`)
- `unattended-upgrades` for security patches only (I-011a)
- Deploy via: SSH → git pull → systemd restart

### US-9.3: systemd Service Management
**Persona: System Administrator**

As an administrator, I want all processes managed by systemd so that the system auto-starts on boot and auto-recovers from process failures.

**Acceptance Criteria:**
- systemd services for: intraday checkpoint process, evening batch, OpenAlgo
- `Restart=on-failure` with watchdog (>60s unresponsive → kill + restart)
- Auto-start on VPS reboot
- Startup self-diagnostic (E-011) runs before entering normal operation

### US-9.4: Per-Module Logging & Observability
**Persona: System Administrator**

As an administrator, I want separate log files per module with error and debug levels so that I can quickly see whether each subsystem is running properly or diagnose issues without wading through unrelated output.

**Acceptance Criteria:**
- Minimum 3-4 separate log files, one per major module:
  - `data.log` — data pipeline (ingestion, validation, BSE calendar)
  - `model.log` — feature engineering, LightGBM training/inference, ranking/MVO
  - `execution.log` — risk engine, OpenAlgo order submission, trade execution, guardrail triggers
  - `scheduler.log` — cron orchestration, checkpoint lifecycle, evening batch lifecycle, heartbeats
- Each log contains both error-level and debug-level entries with ISO timestamps
- Log levels configurable per module (DEBUG, INFO, WARNING, ERROR, CRITICAL) via config file
- Structured format: `[YYYY-MM-DD HH:MM:SS] [MODULE] [LEVEL] message`
- Admin can `tail -f` any single module log independently
- logrotate configured: daily rotation, 30-day retention, compress after 2 days
- Disk usage Telegram alert if NVMe >80%

### US-9.5: Backups
**Persona: System Administrator**

As an administrator, I want automated daily backups so that critical state is protected and recoverable.

**Acceptance Criteria:**
- Daily automated backups after evening batch (~6:15 PM IST) to Hetzner Storage Box
- Backup scope: portfolio state, transaction/audit log, guardrail config, top-N history, latest model artifact
- 30-day backup retention

---

## Epic 10: Backtesting (Phase 1 Gate)

### US-10.1: Backtesting Framework
**Persona: Trader/Owner** (reviews backtest results)

As a trader, I want to run the full strategy against 2 years of historical data so that I have confidence the system is viable before entering Sandbox.

**Acceptance Criteria:**
- Runs complete pipeline: features → LightGBM → ranking → MVO → guardrails → simulated execution
- Models realistic transaction costs: brokerage (~₹20/order), STT (0.1% buy+sell, delivery rate), stamp duty, GST
- Configurable slippage estimate (default: 0.1%, ideally randomized per DC-6)
- Produces report: annualised return (net of costs), Sharpe ratio, max drawdown, win rate, avg holding period, total costs, vs buy-and-hold Sensex
- AI-recommended viability thresholds (per Q9 decision)
- Sandbox mode does not begin until backtest passes minimum viability gate
- Can be re-run after model or feature changes

---

## Epic 11: Telegram Bot Setup (Phase 1)

### US-11.1: Telegram Bot Creation & Integration
**Persona: Trader/Owner**

As a trader, I want a Telegram bot created and integrated so that I receive all system notifications and can perform HITL approvals.

**Acceptance Criteria:**
- New Telegram bot created (per Q7 decision — setup instructions provided)
- Bot token and chat ID securely stored (encrypted)
- Bot can send: daily reports, heartbeats, alerts, guardrail triggers, watchlist approval requests
- Bot can receive: approval/rejection responses for watchlist changes
- Fallback: if Telegram unreachable >5 min, queue alerts locally and retry every 5 min (E-009)

---

## Epic 12: Testing Suite (Phase 1)

### US-12.1: Unit & Integration Tests
**Persona: System Administrator**

As an administrator, I want a comprehensive test suite so that I can validate correctness before deployment and after any code change.

**Acceptance Criteria:**
- Unit tests for all guardrail rules (G-001 through G-013) with boundary conditions
- Unit tests for all feature engineering functions
- Integration test for full evening batch pipeline (data → features → model → ranking → risk → mock execution)
- Integration test for checkpoint pipeline
- All tests must pass before deployment to Hetzner and before any production code change

---

## Future: Epic 13 — Phase 2 (Broker Integration & Live Trading)

> Phase 2 stories are deferred. High-level scope:
> - P2-001: Connect OpenAlgo to Kotak Neo (mock → live credentials)
> - P2-002: HITL approval workflow via Telegram (Trial mode)
> - P2-003: Trial mode with ₹10,000 real funds
> - P2-004: Production mode promotion with ₹50,000
> - P2-005: Production monitoring and alerting
> - P2-006: Broker API live position/price feeds
> - P2-007: Broker TOTP auto-login (X-008)
> - P2-008: Portfolio reconciliation with corporate action auto-adjustment

---

## Story–Persona Mapping (Direct Interaction Only)

| Story | Trader/Owner | System Admin |
|---|---|---|
| US-1.1 Historical Data Bootstrap | ✓ (triggers, monitors) | |
| US-6.1 Seed Watchlist Init | ✓ (configures) | |
| US-6.2 Saturday Screener | ✓ (approves via Telegram) | |
| US-7.1 Daily Telegram Report | ✓ (receives) | |
| US-7.2 Daily Heartbeat | ✓ (monitors) | |
| US-9.1 Local Dev Environment | | ✓ (sets up) |
| US-9.2 VPS Deployment | | ✓ (deploys, hardens) |
| US-9.3 systemd Services | | ✓ (configures) |
| US-9.4 Per-Module Logging | | ✓ (monitors, diagnoses) |
| US-9.5 Backups | | ✓ (configures) |
| US-10.1 Backtesting Framework | ✓ (reviews results) | |
| US-11.1 Telegram Bot Setup | ✓ (uses) | |
| US-12.1 Testing Suite | | ✓ (runs tests) |
