# Application Components — Auto-Trader (AT)

> **Scope**: Phase 1 (Sandbox System). Phase 2 additions noted where relevant.  
> **Architecture**: 12 domain components + service orchestration layer (see services.md).

---

## 1. DataPipeline

**Purpose**: All market data ingestion, validation, and Qlib dataset management.

**Responsibilities**:
- Fetch daily EOD OHLCV from BSE Bhavcopy for all watchlist stocks
- Phase 1 intraday: fetch real intraday prices from AlphaVantage for held symbols + today's baseline proposal symbols
- Phase 2 intraday: fetch real intraday prices from Kotak Neo API
- Fetch Sensex index data (also used for volatility regime computation)
- Fetch historical research data from AlphaVantage
- 2-year historical data bootstrap before first run (Q-006)
- Validate data integrity (D-005/D-005a): reject stale, zero, negative, extreme-deviation prices
- Manage BSE trading calendar (annual holiday list)
- Load raw data into Qlib dataset structure (Q-001 to Q-006)
- Align data by trading date, handle holidays and missing data
- Maintain rolling training windows with minimum 2-year history

**Interfaces**:
- **Provides to** FeatureEngine: Clean, validated, Qlib-structured OHLCV data
- **Provides to** RiskEngine: Sensex level and historical closes (for volatility regime)
- **Provides to** WatchlistManager: Liquidity data for screening
- **Consumes from** orchestration services: Symbol scopes for each run (watchlist for EOD, held + proposed for intraday)
- **Consumes from** ConfigManager: API keys, rate-limit settings, data source config
- **Consumes from** StateManager: Last successful fetch timestamp, bootstrap status

**Requirement Traceability**: D-001 to D-009, Q-001 to Q-006, US-1.1 to US-1.4

---

## 2. FeatureEngine

**Purpose**: Compute technical features from raw price data to produce the feature matrix for LightGBM.

**Responsibilities**:
- Compute all technical features: momentum, volatility, RSI, MACD, volume spikes, moving averages, relative strength vs Sensex
- Design and maintain AI-optimised feature set for BSE swing trading (per Q10)
- Produce feature matrix (stocks × features × dates)
- Ensure deterministic computation (no randomness, no external API calls)
- Handle partial intraday bars at CP1 with validated feature stability (DC-7)
- Provide configurable, extensible feature set without changing core pipeline

**Interfaces**:
- **Provides to** ModelManager: Feature matrix for training and inference
- **Provides to** BacktestEngine: Historical feature matrices
- **Consumes from** DataPipeline: Clean OHLCV data, Sensex data (for relative strength)

**Requirement Traceability**: F-001 to F-005, US-2.1

---

## 3. ModelManager

**Purpose**: LightGBM model lifecycle — training, inference, artifact management, and staleness monitoring.

**Responsibilities**:
- Train LightGBM model on rolling 2-year window (weekly Sunday retrain)
- Run inference to produce alpha scores (expected N-day return) per stock
- Persist model artifacts to disk for crash recovery and versioning
- Retain last 4 weekly model artifacts
- Detect and discard anomalous inference outputs (M-007)
- Monitor model staleness: alert at 14 days, halt buys at 21 days (E-003a)
- Fall back to last known good model on training failure (E-003)
- Log performance metrics (prediction accuracy, Sharpe ratio)

**Interfaces**:
- **Provides to** PortfolioOptimizer: Alpha scores per stock
- **Provides to** ReportGenerator: Model accuracy metrics, model version info
- **Consumes from** FeatureEngine: Feature matrix
- **Consumes from** ConfigManager: Model hyperparameters, prediction horizon (default: 5 days)
- **Consumes from** StateManager: Persisted model artifacts, last retrain timestamp

**Requirement Traceability**: M-001 to M-008, US-2.2, US-2.3

---

## 4. PortfolioOptimizer

**Purpose**: Rank stocks by alpha score and compute optimal position weights using Qlib Mean-Variance Optimization.

**Responsibilities**:
- Rank all watchlist stocks by predicted alpha score
- Select top-N candidates (default: 5, must be ≤ G-002 max positions)
- Compute position weights via Qlib MVO considering volatility and correlations
- Generate trade actions (buy/sell) with specific share quantities
- Apply cold-start ramp logic (S-009): 20%/40%/60%/80%/100% over 5 days
- Identify ranking-based exits (stock dropped out of top-N, respecting M-008 3-day hold)
- Skip re-buy proposals for stocks already held in top-N (S-008)
- Persist top-N membership history (RP-008) for Saturday screener

**Interfaces**:
- **Provides to** RiskEngine: Trade proposals with quantities and weights
- **Provides to** ReportGenerator: Top-N list, ranking changes
- **Provides to** WatchlistManager: Top-N membership history (for removal screening)
- **Consumes from** ModelManager: Alpha scores per stock
- **Consumes from** StateManager: Current holdings, capital, cold-start day count
- **Consumes from** ConfigManager: Top-N count, position weight limits

**Requirement Traceability**: S-001 to S-010, US-3.1, US-3.3

---

## 5. RiskEngine

**Purpose**: Evaluate all guardrails and filter trade proposals to enforce risk limits.

**Responsibilities**:
- **Position sizing** (G-001 to G-003): Max 5%/stock, max 10 positions, max 3%/order
- **Drawdown protection** (G-004, G-005): 3% daily halt, 10% Tier 1 buy-halt (8% hysteresis recovery), 15% Tier 2 full liquidation with 5-day cooling-off
- **Trailing stop-loss** (G-006): 5% below highest-seen price per stock, ratchets up only
- **Volatility regime** (G-007, G-008): Sensex 20-day annualized realized vol >30% halve positions, >45% halt buys
- **Circuit breaker** (G-009): Sensex ≥10% intraday drop → halt all
- **Operational limits** (G-010): Max proposals/day per mode
- **Watchlist limit** (G-011): Max 50 stocks
- Trigger Telegram alerts on every guardrail activation (G-013)
- Log all blocked proposals with full reasoning
- Track highest-seen price per stock for trailing stop-loss (using session high from IM-003)

**Interfaces**:
- **Provides to** ExecutionEngine: Approved trade proposals (guardrail-passed)
- **Provides to** TelegramNotifier: Guardrail trigger alerts
- **Provides to** ReportGenerator: Guardrail activation log (pass/fail per proposal)
- **Consumes from** PortfolioOptimizer: Raw trade proposals
- **Consumes from** DataPipeline: Sensex level, Sensex historical closes, stock prices
- **Consumes from** StateManager: Current holdings, daily P&L, drawdown state, highest-seen prices
- **Consumes from** ConfigManager: All guardrail thresholds (configurable per mode)

**Requirement Traceability**: G-001 to G-013, US-4.1 to US-4.4

---

## 6. ExecutionEngine

**Purpose**: Submit approved trade proposals to OpenAlgo and manage order lifecycle.

**Responsibilities**:
- Submit limit orders at LTP ± 0.3% buffer (buys LTP+, sells LTP−)
- Submit stop-loss exits as market orders
- Track order status: submitted, filled, partially filled, rejected, cancelled
- Auto-cancel unfilled limit orders from CP1 at Mon-1 (IM-006a)
- Validate stock is held before sell orders (X-006)
- Validate liquidity (X-009) before buy orders
- Handle OpenAlgo unreachable: persistent on-disk queue, exponential backoff, re-validate price on retry (E-002)
- Cancel remaining HITL-awaiting proposals at 3:30 PM (IM-010)
- Phase 1: OpenAlgo mock mode; Phase 2: Kotak Neo live via OpenAlgo

**Interfaces**:
- **Provides to** StateManager: Order fill events, position updates
- **Provides to** ReportGenerator: Order execution log (fills, rejections, cancellations)
- **Provides to** TelegramNotifier: Trade execution confirmations
- **Consumes from** RiskEngine: Approved trade proposals
- **Consumes from** DataPipeline: Current stock prices (for limit order pricing)
- **Consumes from** ConfigManager: OpenAlgo connection settings, mock/live mode flag

**Requirement Traceability**: X-001 to X-009, US-3.2

---

## 7. WatchlistManager

**Purpose**: Manage the active watchlist lifecycle — initialisation, screening, and auto-maintenance.

**Responsibilities**:
- Seed watchlist from BSE 100 filtered by X-009 liquidity threshold (~30-40 stocks)
- Saturday screener: evaluate BSE 200/500 candidate pool for additions, flag removals (absent from top-N ≥4 weeks)
- Handle Telegram approval workflow for Sandbox/Trial additions/removals
- Auto-apply with notification in Production mode
- Auto-remove suspended/delisted/circuit-halted stocks >3 days (W-007)
- Manage frozen holdings (excluded from re-ranking, not counted toward G-002, stop-loss continues)
- Trigger 2-year historical data download for new additions (W-009)
- Enforce max 50 watchlist stocks (G-011)

**Interfaces**:
- **Provides to** DataPipeline: Active watchlist (stock symbols)
- **Provides to** TelegramNotifier: Watchlist change approval requests
- **Consumes from** PortfolioOptimizer: Top-N membership history (RP-008)
- **Consumes from** DataPipeline: Liquidity data for screening
- **Consumes from** TelegramNotifier: Approval/rejection responses
- **Consumes from** ConfigManager: Operating mode (for approval workflow routing)
- **Consumes from** StateManager: Current holdings (for frozen holding logic)

**Requirement Traceability**: W-001 to W-009, US-6.1 to US-6.3

---

## 8. TelegramNotifier

**Purpose**: All Telegram Bot communication — sending notifications and receiving HITL approval responses.

**Responsibilities**:
- Send daily reports, heartbeats, alerts, guardrail triggers
- Send watchlist change approval requests (Sandbox/Trial)
- Receive approval/rejection responses for HITL decisions
- 15-minute HITL timeout with auto-action (E-009): auto-execute stop-losses, auto-cancel other proposals
- Queue alerts locally if Telegram unreachable, retry every 5 min
- Manage bot token and chat ID (encrypted storage)
- Handle HITL approval for trade proposals in Trial mode (Phase 2)

**Interfaces**:
- **Provides to** WatchlistManager: Approval/rejection responses
- **Provides to** ExecutionEngine: HITL approval responses (Phase 2 Trial mode)
- **Consumes from** ReportGenerator: Formatted report messages
- **Consumes from** RiskEngine: Guardrail trigger alert data
- **Consumes from** WatchlistManager: Approval request data
- **Consumes from** ConfigManager: Bot token, chat ID, operating mode

**Requirement Traceability**: RP-003 to RP-005, E-009, US-7.1, US-7.2, US-11.1

---

## 9. ReportGenerator

**Purpose**: Format and produce all reports, audit trail entries, and backtest result summaries.

**Responsibilities**:
- Daily portfolio snapshot: holdings, quantities, market values, P&L
- Model prediction accuracy and cumulative performance tracking
- Trades executed/skipped/blocked with full reasoning
- Guardrail activation summary
- Full audit trail per trade: model version, feature snapshot, guardrails pass/fail, portfolio state (RP-005a)
- Backtest result report: annualised return, Sharpe, max DD, win rate, costs, vs Sensex
- Guardrail config change log (G-012)
- Format messages for Telegram delivery

**Interfaces**:
- **Provides to** TelegramNotifier: Formatted report messages
- **Provides to** StateManager: Audit trail entries for persistence
- **Consumes from** StateManager: Portfolio state, transaction history
- **Consumes from** ModelManager: Model version, accuracy metrics
- **Consumes from** RiskEngine: Guardrail activation log
- **Consumes from** ExecutionEngine: Order execution log
- **Consumes from** BacktestEngine: Backtest results

**Requirement Traceability**: RP-001 to RP-008, US-7.1, US-7.3

---

## 10. StateManager

**Purpose**: Encapsulate all SQLite persistence — portfolio state, audit trails, checkpoint logs, and crash recovery.

**Responsibilities**:
- Persist portfolio state: holdings, cash, capital, P&L history
- Persist transaction/order log
- Persist baseline plans, top-N membership history
- Persist checkpoint progress (for crash recovery at market-hours restart)
- Persist guardrail config change history
- Persist drawdown state, highest-seen prices per stock
- Provide crash recovery: load last known state on startup
- Detect completed/missed checkpoints for market-hours restart (E-007)
- All persistence via SQLite (per Q3)
- No other component accesses SQLite directly

**Interfaces**:
- **Provides to** all components: State read/write operations
- **Dependency**: SQLite database files on local disk

**Requirement Traceability**: E-005, E-007, E-010, DC-3, US-8.1

---

## 11. ConfigManager

**Purpose**: Centralised configuration management — operating modes, guardrail settings, API credentials, and change logging.

**Responsibilities**:
- Load configuration from human-readable YAML files (`config/config.yaml`, `config/secrets.yaml`)
- Manage operating mode: Sandbox / Trial / Production
- Provide guardrail thresholds per mode with change logging (G-012)
- Manage API credentials (Phase 1: plaintext `secrets.yaml` with `.gitignore` + file permissions; Phase 2: encrypted storage)
- Provide per-module log level configuration
- Provide model hyperparameters, prediction horizon, top-N count
- Track and log all config changes with timestamp, old/new value, mode affected
- Validate configuration at startup

**Interfaces**:
- **Provides to** all components: Configuration values
- **Provides to** StateManager: Config change events for persistence
- **Dependency**: Config files on disk (YAML)

**Requirement Traceability**: G-012, US-9.4 (log levels)

---

## 12. BacktestEngine

**Purpose**: Run the full strategy against historical data with realistic cost modelling to validate system viability.

**Responsibilities**:
- Run complete pipeline in simulation: features → LightGBM → ranking → MVO → guardrails → simulated execution
- Model transaction costs: brokerage (~₹20/order), STT (0.1% delivery), stamp duty, GST
- Apply configurable slippage estimate (default: 0.1%, ideally randomised per DC-6)
- Produce performance metrics: annualised return (net), Sharpe ratio, max drawdown, win rate, avg holding period, total costs, vs buy-and-hold Sensex
- Apply AI-recommended viability thresholds (per Q9)
- Gate Sandbox entry: backtest must pass minimum viability thresholds
- Re-runnable after model or feature changes

**Interfaces**:
- **Provides to** ReportGenerator: Backtest results for report formatting
- **Consumes from** FeatureEngine: Historical feature matrices
- **Consumes from** ModelManager: Model training and inference (in simulation context)
- **Consumes from** RiskEngine: Guardrail evaluation (in simulation context)
- **Consumes from** ConfigManager: Backtest parameters, viability thresholds

**Requirement Traceability**: P1-017, US-10.1
