# Domain Entities — Unit 1: Foundation & Data

> **Scope**: All shared data types produced and consumed by Unit 1 components  
> **Convention**: Python dataclass style. Types used across units are marked with 🔗.  
> **Storage format where persisted**: SQLite columns → Python types via StateManager

---

## 1. Market Data Types

### 🔗 OHLCVRecord
Daily end-of-day price record. Core data type flowing from DataPipeline to FeatureEngine and Qlib.

```
OHLCVRecord:
  symbol: str               # BSE stock code (e.g., "RELIANCE", "TCS")
  date: date                # Trading date (ISO)
  open: float               # Opening price (₹)
  high: float               # Day's high (₹)
  low: float                # Day's low (₹)
  close: float              # Closing price (₹)
  volume: int               # Shares traded
```

**Produced by**: DataPipeline (from Bhavcopy, AlphaVantage)  
**Consumed by**: FeatureEngine (feature computation), Qlib (dataset storage), StateManager (optional archival)  
**Validation**: See business-rules.md DV-001 through DV-005

---

### 🔗 IntradayBar
Partial or complete intraday bar. Used at checkpoints for real-time pricing.

```
IntradayBar:
  symbol: str               # BSE stock code
  timestamp: datetime       # Bar timestamp (IST)
  open: float
  high: float
  low: float
  close: float              # Current/latest price
  volume: int
```

**Produced by**: DataPipeline (from AlphaVantage intraday in Phase 1, Kotak Neo in Phase 2)  
**Consumed by**: FeatureEngine (partial-bar features at CP1), RiskEngine (stop-loss checks), ExecutionEngine (limit price calculation)  
**Validation**: See business-rules.md DV-006, DV-007

---

## 2. Validation Types

### ValidationResult
Outcome of data validation pass. Separates accepted from rejected records.

```
ValidationResult:
  accepted: dict[str, OHLCVRecord]     # Symbol → validated record
  rejected: list[RejectedRecord]       # Records that failed validation
  acceptance_rate: float               # len(accepted) / total — for E-008 threshold
```

### RejectedRecord
A single record that failed validation, with reason.

```
RejectedRecord:
  symbol: str
  record: OHLCVRecord | IntradayBar    # The original data
  rule: str                            # e.g., "DV-002" or "DV-006"
  reason: str                          # Human-readable: "zero close price"
```

---

## 3. Bootstrap Types

### BootstrapResult
Summary of bootstrap operation across all symbols.

```
BootstrapResult:
  status: BootstrapStatus              # COMPLETE, PARTIAL, FAILED
  per_symbol: dict[str, SymbolBootstrapStatus]
  total_symbols: int
  successful_count: int
  failed_count: int
  duration_seconds: float
```

### SymbolBootstrapStatus
Bootstrap result for a single symbol.

```
SymbolBootstrapStatus:
  symbol: str
  status: str                          # "success", "failed", "gaps_detected"
  records_count: int                   # Number of OHLCV records downloaded
  gap_count: int                       # Consecutive trading day gaps detected
  error_message: str | None            # NULL if success
```

### BootstrapStatus (enum)
```
BootstrapStatus:
  COMPLETE    # All symbols bootstrapped successfully
  PARTIAL     # Some symbols failed (≤20%) — system can proceed with warning
  FAILED      # >20% symbols failed — system blocks startup
```

---

## 4. Configuration Types

### 🔗 AppConfig
Immutable configuration object loaded at startup.

```
AppConfig:
  system: SystemConfig
  data_sources: DataSourceConfig
  scheduling: ScheduleConfig
  model: ModelConfig
  portfolio: PortfolioConfig
  guardrails: GuardrailConfig
  watchlist: WatchlistConfig
  telegram: TelegramConfig
  logging: LoggingConfig
  backup: BackupConfig
```

### SystemConfig
```
SystemConfig:
  mode: str                            # "sandbox", "trial", "production"
  capital: float                       # ₹ — mode-specific
  min_order_value: float               # ₹500 Trial, 0 others
  hitl_required: bool                  # true for Trial
  orders_real: bool                    # false for Sandbox
```

### DataSourceConfig
```
DataSourceConfig:
  alphavantage_base_url: str
  alphavantage_rate_limit_per_minute: int
  bse_bhavcopy_base_url: str
  bse_bhavcopy_url_pattern: str        # Date template for download URL
  kotak_neo_enabled: bool
  kotak_neo_base_url: str | None
```

### ScheduleConfig
```
ScheduleConfig:
  cp1_time: str                        # "09:30"
  monitoring_checkpoints: list[str]    # ["10:30", "11:30", "12:30", "13:30", "14:30"]
  evening_batch_time: str              # "16:00"
  pre_market_time: str                 # "09:10"
  heartbeat_trading_day: str           # "09:15"
  heartbeat_non_trading: str           # "12:00"
  weekly_screener: str                 # "saturday 10:00"
  weekly_retrain: str                  # "sunday 02:00"
  timezone: str                        # "Asia/Kolkata"
```

### ModelConfig
```
ModelConfig:
  horizon_days: int                    # Default: 5
  retrain_day: str                     # "sunday"
  staleness_warn_days: int             # 14
  staleness_halt_days: int             # 21
  min_training_years: int              # 2
```

### PortfolioConfig
```
PortfolioConfig:
  top_n: int                           # Default: 5
  cold_start_ramp: list[float]         # [0.20, 0.40, 0.60, 0.80, 1.00]
  min_holding_days: int                # 3
```

### 🔗 GuardrailConfig
All guardrail threshold values. Loaded from YAML, overridable per mode.

```
GuardrailConfig:
  g001_max_pct_per_stock: float        # 0.05
  g002_max_positions: int              # 10
  g003_max_pct_per_order: float        # 0.03
  g004_daily_loss_halt_pct: float      # 0.03
  g005a_tier1_drawdown_pct: float      # 0.10
  g005b_tier2_drawdown_pct: float      # 0.15
  g005_tier1_hysteresis_pct: float     # 0.08
  g005b_cooling_days: int              # 5
  g006_trailing_stop_pct: float        # 0.05
  g007_vol_halve_threshold: float      # 0.30
  g008_vol_halt_threshold: float       # 0.45
  g009_circuit_breaker_pct: float      # 0.10
  g010_max_proposals_production: int   # 10
  g010_max_proposals_trial: int        # 3
  g011_max_watchlist: int              # 50
```

### WatchlistConfig
```
WatchlistConfig:
  seed_index: str                      # "BSE100"
  candidate_pool: str                  # "BSE200"
  liquidity_threshold_lakhs: float     # 50
  removal_absent_weeks: int            # 4
```

### TelegramConfig
```
TelegramConfig:
  enabled: bool
  heartbeat_enabled: bool
  fallback_retry_minutes: int          # 5
```

### LoggingConfig
```
LoggingConfig:
  levels: dict[str, str]              # { "data": "INFO", "model": "INFO", ... }
  rotation: str                        # "daily"
  retention_days: int                  # 30
  compress_after_days: int             # 2
```

### BackupConfig
```
BackupConfig:
  enabled: bool
  time: str                            # "18:15"
  retention_days: int                  # 30
```

---

## 5. State & Persistence Types

### 🔗 Holding
Current portfolio position for a single stock. Maps to `holdings` table row.

```
Holding:
  symbol: str
  quantity: int
  avg_entry_price: float               # ₹
  entry_date: date
  highest_seen_price: float            # ₹ — for trailing stop-loss (G-006)
  frozen: bool                         # True if suspended/delisted (W-007)
  mode: str                            # "sandbox", "trial", "production"
  updated_at: datetime
```

### 🔗 Order
Order record. Maps to `orders` table row.

```
Order:
  id: int
  symbol: str
  action: str                          # "BUY" or "SELL"
  quantity: int
  order_type: str                      # "LIMIT" or "MARKET"
  limit_price: float | None
  status: OrderStatus
  source: str                          # "ranking", "stop_loss", "drawdown", "manual"
  checkpoint: str                      # "CP1", "Mon-1", ..., "evening_batch"
  trade_date: date                     # Explicit trade date (ISO)
  submitted_at: datetime
  filled_at: datetime | None
  fill_price: float | None
  fill_quantity: int | None
  broker_order_id: str | None
  retry_count: int
  error_message: str | None
  mode: str                            # "sandbox", "trial", "production"
```

### OrderStatus (enum)
```
OrderStatus:
  SUBMITTED     # Sent to broker / mock engine
  FILLED        # Fully filled
  PARTIAL       # Partially filled (Phase 2 edge case)
  REJECTED      # Broker rejected
  CANCELLED     # User or system cancelled (e.g., unfilled at Mon-1)
```

**Valid transitions** (see business-rules.md SP-004):
- SUBMITTED → FILLED | PARTIAL | REJECTED | CANCELLED
- PARTIAL → FILLED | CANCELLED
- FILLED, REJECTED, CANCELLED are terminal

### 🔗 BaselinePlan
Evening batch output — the plan for the next trading day. Maps to `baseline_plans` table.

```
BaselinePlan:
  id: int
  plan_date: date
  ranked_stocks: list[RankedStock]     # JSON-serialized
  trade_proposals: list[TradeProposal] # JSON-serialized
  vol_regime: VolatilityRegimeStatus
  plan_status: PlanStatus
  model_version: str
  created_at: datetime
```

### PlanStatus (enum)
```
PlanStatus:
  CREATED       # Generated by evening batch
  EXECUTED      # CP1 acted on this plan
  STALE         # Data issue flagged (plan_stale_flag)
  SUPERSEDED    # Replaced by newer plan
```

### 🔗 DrawdownState
Singleton drawdown tracking. Maps to `drawdown_state` table.

```
DrawdownState:
  peak_capital: float                  # ₹ — highest ever portfolio value
  daily_start_value: float             # ₹ — portfolio value at start of today
  tier: DrawdownTier
  cooling_off_start: date | None       # Start date of cooling-off period
  cooling_off_remaining: int | None    # Days remaining
  updated_at: datetime
```

### DrawdownTier (enum)
```
DrawdownTier:
  NONE          # Normal operation
  DAILY_HALT    # G-004 triggered — 3% daily loss, halt for rest of day
  TIER1         # G-005a — 10% total drawdown, buy-halt with hysteresis at 8%
  TIER2         # G-005b — 15% total drawdown, full liquidation + 5-day cooling
```

### CheckpointLogEntry
Maps to `checkpoint_log` table.

```
CheckpointLogEntry:
  id: int
  checkpoint_date: date
  checkpoint_name: str                 # "pre_market", "CP1", "Mon-1".."Mon-5", "close"
  status: CheckpointStatus
  started_at: datetime
  completed_at: datetime | None
  duration_seconds: float | None
  error_message: str | None
```

### CheckpointStatus (enum)
```
CheckpointStatus:
  STARTED
  COMPLETED
  FAILED
  TIMEOUT
  SKIPPED       # E.g., skipped due to E-008 batch rejection
```

### AuditEntry
Per-trade decision record. Maps to `audit_trail` table.

```
AuditEntry:
  id: int
  trade_date: date
  symbol: str
  action: str                          # "BUY", "SELL", "SKIP"
  model_version: str
  alpha_score: float | None
  rank: int | None
  feature_snapshot: dict[str, float]   # Key feature values
  guardrail_results: list[GuardrailResult]  # JSON-serialized
  final_decision: str                  # "APPROVED", "BLOCKED", "MODIFIED"
  reason: str | None
  checkpoint: str
  created_at: datetime
```

### ConfigChangeEntry
Maps to `config_changes` table.

```
ConfigChangeEntry:
  id: int
  changed_at: datetime
  field_name: str
  old_value: str
  new_value: str
  changed_by: str                      # "system" or "admin"
  reason: str | None
```

### TopNSnapshot
Maps to `top_n_snapshots` table.

```
TopNSnapshot:
  id: int
  snapshot_date: date
  symbols: list[str]                   # JSON-serialized
  created_at: datetime
```

### WatchlistEntry
Maps to `watchlist` table row.

```
WatchlistEntry:
  id: int
  symbol: str
  added_date: date
  removed_date: date | None
  status: str                          # "active", "removed", "frozen"
  source: str                          # "seed", "screener", "manual"
  reason: str | None
  updated_at: datetime
```

---

## 6. Recovery & Diagnostic Types

### RecoveryState
Produced by `StateManager.detect_recovery_state()`. Consumed by StartupService (Unit 4).

```
RecoveryState:
  needs_bootstrap: bool
  completed_checkpoints: list[str]     # Checkpoint names completed today
  pending_orders: int                  # Count of SUBMITTED orders for today
  drawdown_tier: DrawdownTier
  last_eod_date: date | None           # Last successful evening batch date
  is_plan_stale: bool
```

### DiagnosticResult
Produced by `StateManager.run_diagnostics()`. Consumed by StartupService.

```
DiagnosticResult:
  checks: dict[str, DiagnosticCheck]   # Check name → result
  overall_status: str                  # "pass", "fail", "warning"
  timestamp: datetime
```

### DiagnosticCheck
```
DiagnosticCheck:
  name: str                            # e.g., "database_integrity", "schema_version"
  status: str                          # "pass", "fail", "warning"
  message: str                         # Description of check result
  duration_ms: float | None            # How long the check took
```

---

## 7. Cross-Unit Types (produced elsewhere, referenced here)

These types are defined in Unit 1 (in `src/models/types.py`) but fully populated by later units:

### 🔗 TradeProposal
Defined in Unit 1 types module, populated by PortfolioOptimizer (Unit 2), consumed by RiskEngine/ExecutionEngine (Unit 3).

```
TradeProposal:
  symbol: str
  action: str                          # "BUY" or "SELL"
  quantity: int
  reason: str                          # "ranking", "stop_loss", "drawdown"
  source: str                          # Which component generated this
```

### 🔗 RankedStock
```
RankedStock:
  symbol: str
  alpha_score: float
  rank: int
```

### 🔗 VolatilityRegimeStatus (enum)
Produced by RiskEngine (Unit 3), stored in BaselinePlan, used in services.

```
VolatilityRegimeStatus:
  NORMAL        # Sensex 20-day annualized vol ≤ 30%
  HALVE         # G-007: vol > 30% — halve all position sizes
  HALT_BUYS     # G-008: vol > 45% — no new buys, sells/exits only
```

### 🔗 GuardrailResult
```
GuardrailResult:
  guardrail_id: str                    # "G-001", "G-002", etc.
  passed: bool
  reason: str                          # Human-readable explanation
  values: dict[str, float] | None      # Relevant thresholds/actuals
```

### 🔗 PortfolioState
Aggregated portfolio snapshot used by RiskEngine.

```
PortfolioState:
  holdings: dict[str, Holding]         # Symbol → Holding
  cash: float                          # Available cash (₹)
  total_capital: float                 # cash + sum(holding values)
  peak_capital: float                  # Historical peak (for drawdown)
  daily_start_value: float             # Value at market open today
```

### 🔗 MarketSnapshot
Current market state passed to RiskEngine at each checkpoint.

```
MarketSnapshot:
  prices: dict[str, float]            # Symbol → current price (₹)
  sensex: float                        # Current Sensex level
  sensex_closes_20d: list[float]       # Last 20 trading days of Sensex closes
  sensex_change_pct: float             # Intraday Sensex change % (for circuit breaker)
```

---

## 8. Type Location

All types defined in `src/models/types.py` as Python dataclasses (or named tuples for simple value types). Enums use `enum.Enum`.

Types marked 🔗 are shared across units and must be stable — changes require cross-unit impact assessment.
