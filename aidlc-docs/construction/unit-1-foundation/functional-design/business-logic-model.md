# Business Logic Model — Unit 1: Foundation & Data

> **Scope**: ConfigManager, StateManager, DataPipeline, Per-Module Logging  
> **Design answers**: All from unit-1-functional-design-plan.md (Q1–Q12)  
> **Technology-agnostic**: Business logic only — no infrastructure, deployment, or NFR concerns

---

## 1. ConfigManager

### 1.1 Configuration Loading

**Config file**: `config/config.yaml` (YAML, per Q1)

**Load flow**:
1. Read `config/config.yaml` from filesystem
2. Parse YAML → in-memory dict
3. Validate against expected schema (see §1.2)
4. If validation fails → raise with descriptive error listing all failures, system does not start
5. Freeze config as immutable object — no runtime mutation except via `switch_mode()`
6. Log loaded config summary (mode, watchlist size, checkpoint count) at INFO level — **never log credential values**

**Schema structure** (logical sections):
```
system:
  mode: sandbox | trial | production
  capital: float
  min_order_value: float          # ₹500 for Trial, 0 for others

data_sources:
  alphavantage:
    base_url: str
    rate_limit_per_minute: int    # AlphaVantage: 5/min free tier
    bootstrap_output_size: full
  bse_bhavcopy:
    base_url: str
    archive_url_pattern: str      # Date-templated download URL
  kotak_neo:
    enabled: bool                 # false in Phase 1
    base_url: str

scheduling:
  cp1_time: "09:30"
  monitoring_checkpoints: ["10:30", "11:30", "12:30", "13:30", "14:30"]
  evening_batch_time: "16:00"
  pre_market_time: "09:10"
  heartbeat_trading_day: "09:15"
  heartbeat_non_trading: "12:00"
  weekly_screener: "saturday 10:00"
  weekly_retrain: "sunday 02:00"
  timezone: "Asia/Kolkata"

model:
  horizon_days: 5
  retrain_day: sunday
  staleness_warn_days: 14
  staleness_halt_days: 21
  min_training_years: 2

portfolio:
  top_n: 5
  cold_start_ramp: [0.20, 0.40, 0.60, 0.80, 1.00]
  min_holding_days: 3

guardrails:
  g001_max_pct_per_stock: 0.05
  g002_max_positions: 10
  g003_max_pct_per_order: 0.03
  g004_daily_loss_halt_pct: 0.03
  g005a_tier1_drawdown_pct: 0.10
  g005b_tier2_drawdown_pct: 0.15
  g005_tier1_hysteresis_pct: 0.08
  g005b_cooling_days: 5
  g006_trailing_stop_pct: 0.05
  g007_vol_halve_threshold: 0.30
  g008_vol_halt_threshold: 0.45
  g009_circuit_breaker_pct: 0.10
  g010_max_proposals_production: 10
  g010_max_proposals_trial: 3
  g011_max_watchlist: 50

watchlist:
  seed_index: "BSE100"
  candidate_pool: "BSE200"
  liquidity_threshold_lakhs: 50
  removal_absent_weeks: 4

telegram:
  enabled: bool
  heartbeat_enabled: bool
  fallback_retry_minutes: 5

logging:
  levels:
    data: INFO
    model: INFO
    execution: INFO
    scheduler: INFO
  rotation: daily
  retention_days: 30
  compress_after_days: 2

backup:
  enabled: bool
  time: "18:15"
  retention_days: 30
```

### 1.2 Schema Validation

**Validation rules** (applied at load time):
1. `system.mode` must be one of: `sandbox`, `trial`, `production`
2. `system.capital` must be > 0
3. All guardrail percentages must be 0 < value < 1
4. `guardrails.g002_max_positions` must be ≥ `portfolio.top_n`
5. `scheduling.monitoring_checkpoints` must be a non-empty list of valid HH:MM strings
6. All time strings must be valid 24-hour HH:MM format
7. `model.min_training_years` must be ≥ 1
8. `portfolio.cold_start_ramp` must be ascending list ending at 1.0
9. `portfolio.min_holding_days` must be ≥ 1
10. `watchlist.liquidity_threshold_lakhs` must be > 0
11. `logging.levels.*` must be valid Python log level names

**Validation output**: List of `(field_path, error_message)` tuples. All errors collected before raising (not fail-fast on first error).

### 1.3 Mode Management

**Modes**: Sandbox, Trial, Production — each overrides specific guardrail values.

**`get_mode() -> str`**: Returns current mode string.

**`switch_mode(new_mode: str) -> None`**:
1. Validate `new_mode` is one of the three valid modes
2. Log mode change to `config_changes` table via StateManager (G-012)
3. Update in-memory config with mode-specific overrides
4. Emit Telegram notification of mode change

**Mode-specific overrides** (hardcoded per mode, not in YAML):
| Setting | Sandbox | Trial | Production |
|---|---|---|---|
| `capital` | 50000 (simulated) | 10000 (real) | 50000 (real) |
| `g010_max_proposals` | 10 | 3 | 10 |
| `min_order_value` | 0 | 500 | 0 |
| `hitl_required` | false | true | false |
| `orders_real` | false | true | true |

### 1.4 Credential Access

**Credential file**: `config/secrets.yaml` — excluded from git via `.gitignore` (per Q2, Phase 1 approach).

**Structure**:
```
alphavantage_api_key: str
telegram_bot_token: str
telegram_chat_id: str
```

**`get_credential(name: str) -> str`**:
1. Read `secrets.yaml` (cached in memory after first read)
2. Return value for given key
3. Raise if key not found
4. **Never log the returned value** — log only that a credential was accessed

**Phase 2 addition**: Replace file-based credentials with Fernet-encrypted storage using machine-derived key.

### 1.5 Change Logging (G-012)

Every guardrail config change is recorded:
- **What**: Field name, old value, new value
- **When**: ISO timestamp
- **Who**: "system" (mode switch) or "admin" (manual edit)
- **Persisted via**: StateManager → `config_changes` table

---

## 2. StateManager

### 2.1 Database Initialization

**Database**: Single SQLite file at `data/autotrader.db` (per Q3, Q4).

**Initialization flow**:
1. Check if database file exists
2. If not: create file, execute all `CREATE TABLE` DDL statements, set `bootstrap_complete = false` in `system_state`
3. If exists: validate schema version matches expected (simple version integer in `system_state`)
4. Enable WAL mode for concurrent reads during checkpoints
5. Enable foreign keys (`PRAGMA foreign_keys = ON`)

### 2.2 SQLite Schema (9 Tables — per Q4)

**Table 1: `holdings`** — Current portfolio positions
```
holdings:
  id: INTEGER PRIMARY KEY AUTOINCREMENT
  symbol: TEXT NOT NULL
  quantity: INTEGER NOT NULL
  avg_entry_price: REAL NOT NULL
  entry_date: TEXT NOT NULL          # ISO date
  highest_seen_price: REAL NOT NULL  # For trailing stop-loss G-006
  frozen: INTEGER DEFAULT 0          # 1 if stock suspended/delisted (W-007)
  updated_at: TEXT NOT NULL          # ISO timestamp
  UNIQUE(symbol)                     # One row per held symbol
```

**Table 2: `orders`** — Full order history
```
orders:
  id: INTEGER PRIMARY KEY AUTOINCREMENT
  symbol: TEXT NOT NULL
  action: TEXT NOT NULL              # BUY or SELL
  quantity: INTEGER NOT NULL
  order_type: TEXT NOT NULL          # LIMIT or MARKET
  limit_price: REAL                  # NULL for MARKET orders
  status: TEXT NOT NULL              # SUBMITTED, FILLED, PARTIAL, REJECTED, CANCELLED
  source: TEXT NOT NULL              # ranking, stop_loss, drawdown, manual
  checkpoint: TEXT NOT NULL          # CP1, Mon-1..Mon-5, evening_batch
  submitted_at: TEXT NOT NULL
  filled_at: TEXT
  fill_price: REAL
  fill_quantity: INTEGER
  broker_order_id: TEXT              # OpenAlgo order ref (NULL in Sandbox mock)
  retry_count: INTEGER DEFAULT 0
  error_message: TEXT
```

**Table 3: `baseline_plans`** — Evening batch plans
```
baseline_plans:
  id: INTEGER PRIMARY KEY AUTOINCREMENT
  plan_date: TEXT NOT NULL UNIQUE    # ISO date
  ranked_stocks: TEXT NOT NULL       # JSON list of {symbol, alpha_score, rank}
  trade_proposals: TEXT NOT NULL     # JSON list of TradeProposal
  vol_regime: TEXT NOT NULL          # NORMAL, HALVE, HALT_BUYS
  plan_status: TEXT NOT NULL         # CREATED, EXECUTED, STALE, SUPERSEDED
  model_version: TEXT NOT NULL
  created_at: TEXT NOT NULL
```

**Table 4: `checkpoint_log`** — Checkpoint completion tracking
```
checkpoint_log:
  id: INTEGER PRIMARY KEY AUTOINCREMENT
  checkpoint_date: TEXT NOT NULL     # ISO date
  checkpoint_name: TEXT NOT NULL     # pre_market, CP1, Mon-1..Mon-5, close
  status: TEXT NOT NULL              # STARTED, COMPLETED, FAILED, TIMEOUT, SKIPPED
  started_at: TEXT NOT NULL
  completed_at: TEXT
  duration_seconds: REAL
  error_message: TEXT
```

**Table 5: `audit_trail`** — Per-trade decision audit
```
audit_trail:
  id: INTEGER PRIMARY KEY AUTOINCREMENT
  trade_date: TEXT NOT NULL
  symbol: TEXT NOT NULL
  action: TEXT NOT NULL              # BUY, SELL, SKIP
  model_version: TEXT NOT NULL
  alpha_score: REAL
  rank: INTEGER
  feature_snapshot: TEXT             # JSON of key feature values at decision time
  guardrail_results: TEXT NOT NULL   # JSON list of {guardrail_id, passed, reason}
  final_decision: TEXT NOT NULL      # APPROVED, BLOCKED, MODIFIED
  reason: TEXT                       # Human-readable summary
  checkpoint: TEXT NOT NULL
  created_at: TEXT NOT NULL
```

**Table 6: `drawdown_state`** — Drawdown tracking
```
drawdown_state:
  id: INTEGER PRIMARY KEY             # Always 1 (singleton)
  peak_capital: REAL NOT NULL
  daily_start_value: REAL NOT NULL
  tier: TEXT NOT NULL DEFAULT 'NONE'  # NONE, DAILY_HALT, TIER1, TIER2
  cooling_off_start: TEXT             # ISO date, NULL if not cooling
  cooling_off_remaining: INTEGER      # Days, NULL if not cooling
  updated_at: TEXT NOT NULL
```

**Table 7: `config_changes`** — Guardrail config change log (G-012)
```
config_changes:
  id: INTEGER PRIMARY KEY AUTOINCREMENT
  changed_at: TEXT NOT NULL
  field_name: TEXT NOT NULL
  old_value: TEXT NOT NULL
  new_value: TEXT NOT NULL
  changed_by: TEXT NOT NULL           # "system" or "admin"
  reason: TEXT
```

**Table 8: `top_n_snapshots`** — Top-N membership history (RP-008)
```
top_n_snapshots:
  id: INTEGER PRIMARY KEY AUTOINCREMENT
  snapshot_date: TEXT NOT NULL
  symbols: TEXT NOT NULL              # JSON list of symbols
  created_at: TEXT NOT NULL
```

**Table 9: `system_state`** — Key-value store for miscellaneous state
```
system_state:
  key: TEXT PRIMARY KEY
  value: TEXT NOT NULL
  updated_at: TEXT NOT NULL
```

**Predefined keys in `system_state`**:
- `bootstrap_complete`: "true"/"false"
- `schema_version`: integer as string
- `last_eod_fetch_date`: ISO date
- `last_bootstrap_date`: ISO date
- `last_retrain_date`: ISO date
- `last_backup_date`: ISO date
- `current_mode`: "sandbox"/"trial"/"production"
- `plan_stale_flag`: "true"/"false"

### 2.3 CRUD Operations

Each table has standard operations. The StateManager exposes purpose-specific methods rather than raw SQL:

**Holdings**:
- `upsert_holding(symbol, quantity, avg_entry_price, entry_date, highest_seen_price)` — Insert or update by symbol
- `remove_holding(symbol)` — Delete row (on full sell)
- `get_holding(symbol) -> Holding | None`
- `get_all_holdings() -> list[Holding]`
- `update_highest_seen(symbol, price)` — Update trailing stop reference (called each checkpoint with current price)
- `set_frozen(symbol, frozen: bool)` — Mark as frozen/unfrozen (W-007)

**Orders**:
- `insert_order(order_data) -> int` — Returns order ID
- `update_order_status(order_id, status, fill_price, fill_quantity, filled_at)` — Lifecycle update
- `get_pending_orders(date) -> list[Order]` — WHERE status = 'SUBMITTED' AND DATE(submitted_at, 'localtime') = date
- `get_orders_by_date(date) -> list[Order]`
- `get_order_count_today() -> int` — For G-010 enforcement

**Baseline Plans**:
- `save_plan(plan_data)` — Insert new plan
- `get_latest_plan() -> BaselinePlan | None`
- `mark_plan_executed(plan_id)`
- `mark_plan_stale(plan_id)`

**Checkpoint Log**:
- `log_checkpoint_start(date, name) -> int` — Returns log ID
- `log_checkpoint_end(log_id, status, duration, error)` — Finalize
- `get_completed_checkpoints(date) -> list[str]` — For crash recovery (which CPs ran today)

**Audit Trail**:
- `log_decision(audit_data)` — Insert decision audit row
- `get_decisions_by_date(date) -> list[AuditEntry]`

**Drawdown State**:
- `get_drawdown_state() -> DrawdownState`
- `update_drawdown_state(peak, daily_start, tier, cooling_start, cooling_remaining)`
- `reset_daily_start(value)` — Called at market open each day

**Config Changes**:
- `log_config_change(field, old_val, new_val, changed_by, reason)`
- `get_config_history(field: str | None) -> list[ConfigChange]`

**Top-N Snapshots**:
- `save_top_n_snapshot(date, symbols)` — Insert daily snapshot
- `get_top_n_history(weeks: int) -> list[TopNSnapshot]` — For Saturday screener (4-week absence check)

**System State**:
- `get_state(key) -> str | None`
- `set_state(key, value)` — Upsert

### 2.4 Crash Recovery State Detection

**Purpose**: On startup, determine what state the system was in when it last stopped.

**`detect_recovery_state() -> RecoveryState`**:
1. Read `system_state` key `bootstrap_complete` — if "false", recovery state = `NEEDS_BOOTSTRAP`
2. Read today's `checkpoint_log` entries — determine which checkpoints completed
3. Read `orders` with `status = SUBMITTED` and today's date — count unresolved orders
4. Read `drawdown_state` — check if cooling-off is active
5. Return `RecoveryState`:
   - `completed_checkpoints`: list of checkpoint names completed today
   - `pending_orders`: count of unresolved orders
   - `drawdown_tier`: current tier
   - `needs_bootstrap`: bool
   - `last_eod_date`: last successful evening batch date
   - `is_plan_stale`: bool (plan exists but wasn't executed)

### 2.5 Self-Diagnostic (E-011)

**`run_diagnostics() -> DiagnosticResult`**:

Run on every system startup before entering normal operation:

1. **Database integrity**: `PRAGMA integrity_check` — must return "ok"
2. **Schema version match**: `system_state.schema_version` matches code expected version
3. **Bootstrap status**: Check if bootstrap is complete
4. **Model artifact exists**: Check filesystem for model file (if post-unit-2)
5. **Config loadable**: ConfigManager can parse config.yaml without errors
6. **Credential access**: All required credentials present in secrets.yaml
7. **Disk space**: Check NVMe usage < 80% (E-004 threshold for Telegram alert)
8. **Log files writable**: Test write to each log file

**Result**: `DiagnosticResult` with per-check pass/fail/warning status and overall status.

**On failure**: System logs diagnostic results, sends Telegram alert (if possible), and refuses to start.

---

## 3. DataPipeline

### 3.1 Architecture Overview

DataPipeline uses three data source adapters behind a unified interface:

```
DataPipeline
├── AlphaVantageAdapter   — Historical bootstrap + Sensex + intraday prices (Phase 1)
├── BhavCopyAdapter       — Daily EOD OHLCV
└── KotakNeoAdapter       — Real-time intraday (Phase 2, disabled in Phase 1)
```

Each adapter handles its own rate limiting, error handling, and response parsing. DataPipeline coordinates them and manages validation + Qlib integration.

### 3.2 AlphaVantage Adapter

**Rate limiting**: Max 5 calls/minute on free tier. Implement a call-level throttle: track call timestamps, sleep if threshold approached. Space calls ≥12 seconds apart.

**`fetch_daily_history(symbol: str, outputsize: str = "full") -> list[OHLCVRecord]`**:
1. Call `TIME_SERIES_DAILY` endpoint with `symbol` and `outputsize`
2. Parse JSON response → list of OHLCVRecord
3. Handle error responses (invalid symbol, rate limit exceeded, API key issue)
4. Return sorted by date ascending

**`fetch_sensex_daily() -> list[OHLCVRecord]`**:
1. Call `TIME_SERIES_DAILY` with canonical Sensex symbol `BSE:SENSEX`
2. If canonical symbol fails, try fallback symbols in order: `^BSESN`, `SENSEX`
3. Parse → list of OHLCVRecord
4. Return sorted by date ascending

**`fetch_intraday(symbol: str, interval: str = "5min") -> list[IntradayBar]`**:
1. Call `TIME_SERIES_INTRADAY` with symbol and interval
2. Parse → list of IntradayBar
3. For checkpoint usage: extract latest bar as current partial bar
4. Rate limit: 5/min constraint means fetching 50 stocks takes ~10 minutes — batch carefully

**Intraday rate limit challenge (Phase 1)**:
- 50 stocks × 1 call each = 50 calls → 10 minutes at 5/min → exceeds 10-min checkpoint timeout
- **Mitigation**: Use `TIME_SERIES_INTRADAY` with `outputsize=compact` (latest 100 data points). Consider fetching only held positions + top-N candidates (typically ≤10-15 symbols) instead of full watchlist for intraday. Full watchlist only needed for Evening Batch (which has more time).
- **Decision**: At CP1 and monitoring checkpoints, fetch intraday only for: (a) held positions, (b) today's baseline plan proposals. This keeps call count at ~10-15, completing in ~3 minutes.

### 3.3 BSE Bhavcopy Adapter

**`fetch_bhavcopy(date: date) -> dict[str, OHLCVRecord]`**:
1. Construct URL from date-based pattern (BSE publishes Bhavcopy at a predictable URL with date interpolation)
2. Download CSV file (~2-3 MB)
3. Parse CSV → all records
4. Filter to requested symbols (from watchlist) + any additional symbols needed for screening
5. Convert to OHLCVRecord format
6. Return dict keyed by symbol

**Daily ongoing flow** (called by EveningBatchService):
1. Check BSE calendar — skip if non-trading day
2. Download today's Bhavcopy CSV
3. Filter to watchlist symbols
4. Validate each record (§3.6)
5. Store validated records
6. Update `system_state.last_eod_fetch_date`

**Bootstrap flow** — Not used for bootstrap (AlphaVantage handles it per Q7/D answer). Bhavcopy is daily ongoing only.

### 3.4 Kotak Neo Adapter (Phase 2)

**Phase 1**: Disabled. All intraday data comes from AlphaVantage intraday endpoint (per Q6/A answer — user wants real data in Phase 1).

**Phase 2**: Swap in Kotak Neo API for real-time prices and live order execution. Interface unchanged — `fetch_intraday_prices(symbols) -> dict[str, IntradayBar]`.

### 3.5 Bootstrap Flow (One-Time, Pre-Sandbox)

**Purpose**: Download 2-year OHLCV history for all seed watchlist stocks + Sensex before first run.

**`bootstrap_history(symbols: list[str], years: int = 2) -> BootstrapResult`**:
1. Check `system_state.bootstrap_complete` — if "true", skip (idempotent)
2. For each symbol in `symbols`:
   a. Call AlphaVantage `TIME_SERIES_DAILY` with `outputsize=full`
   b. Filter to last `years` years of data
   c. Validate all records (§3.6)
   d. Track per-symbol status (success, failed, gap_count)
   e. Respect rate limit (12-second spacing)
3. Fetch Sensex history via `fetch_sensex_daily()`
4. Load all validated data into Qlib dataset (§3.7)
5. Set `system_state.bootstrap_complete = "true"`
6. Set `system_state.last_bootstrap_date` = today
7. Return `BootstrapResult` with per-symbol status

**Estimated duration**: 50 stocks + Sensex = 51 calls × 12s spacing = ~10.2 minutes. Acceptable for one-time operation.

**Gap handling**: If a symbol has >3 consecutive trading day gaps, flag it in the result but still include the data. The FeatureEngine will handle gaps with forward-fill.

### 3.6 Data Validation (D-005, D-005a)

**EOD Validation (D-005)** — Applied to Bhavcopy and AlphaVantage daily data:
1. **Completeness**: OHLCV fields must all be present and non-null
2. **Zero check**: Reject if any of O/H/L/C = 0
3. **Negative check**: Reject if any of O/H/L/C/V < 0
4. **OHLC consistency**: High ≥ max(Open, Close), Low ≤ min(Open, Close)
5. **Extreme deviation**: If |close_today / close_yesterday - 1| > 0.20 (20% daily move), flag for review but do not auto-reject (BSE circuit limits allow 20% moves for some stocks)

**Intraday Validation (D-005a)** — Applied to AlphaVantage intraday and Kotak Neo data:
1. All EOD rules (1-4) apply
2. **Staleness**: Reject if bar timestamp is >15 minutes old (at checkpoint time)
3. **Extreme deviation**: Reject if |current_price / prev_close - 1| > 0.20

**Batch rejection rule (E-008)**:
- If >30% of fetched symbols fail validation in a single checkpoint → skip entire checkpoint
- Log which symbols failed and why
- Send Telegram alert

**Output**: `ValidationResult` with two lists — accepted records and rejected records (each with rejection reason).

### 3.7 Qlib Dataset Management

**Standard Qlib binary format** (per Q9):

**Directory structure**:
```
data/qlib_dataset/
├── instruments/
│   ├── all.txt                  # Symbol list with date ranges
├── calendars/
│   ├── day.txt                  # Trading day list
├── features/
│   ├── SYMBOL1/
│   │   ├── open.day.bin
│   │   ├── high.day.bin
│   │   ├── low.day.bin
│   │   ├── close.day.bin
│   │   └── volume.day.bin
│   ├── SYMBOL2/
│   │   └── ...
│   └── SENSEX/
│       ├── close.day.bin        # Sensex close (for relative strength + vol regime)
│       └── ...
```

**`load_into_qlib(raw_data: dict[str, list[OHLCVRecord]]) -> None`**:
1. Convert raw OHLCVRecords to pandas DataFrames (per symbol)
2. Align by BSE trading calendar dates
3. Forward-fill gaps ≤3 days; leave longer gaps as NaN (FeatureEngine handles NaN)
4. Write to Qlib binary format using `dump_bin.py` conversion utilities
5. Update `instruments/all.txt` with symbol date ranges
6. Update `calendars/day.txt` with trading days

**Daily append** (after Evening Batch EOD fetch):
1. Append new day's data to existing Qlib binary files
2. Update calendar file
3. Update instrument date ranges if needed

**Rolling window**: Keep last 2+ years of data. No pruning in Phase 1 (disk space is abundant on 160 GB NVMe).

### 3.8 BSE Trading Calendar

**Source**: `config/bse_holidays.yaml` — manually maintained annual list of BSE holidays.

**Structure**:
```
year: 2026
holidays:
  - date: "2026-01-26"
    name: "Republic Day"
  - date: "2026-03-10"
    name: "Holi"
  ...
```

**`get_trading_calendar(year: int) -> list[date]`**:
1. Generate all weekdays for the given year
2. Subtract BSE holidays from the list
3. Return sorted list of trading days

**`is_trading_day(check_date: date) -> bool`**:
1. If weekend → False
2. If in holidays list → False
3. Otherwise → True

**Calendar refresh**: Loaded at startup. New year's calendar must be added manually to `bse_holidays.yaml` before January 1.

### 3.9 Sensex History for Volatility Regime

**`fetch_sensex_history(days: int = 20) -> list[float]`**:
1. Query Qlib dataset for last `days` trading days of Sensex close values
2. If Qlib dataset has sufficient data → return from local cache
3. If insufficient (e.g., first day after bootstrap) → fetch from AlphaVantage
4. Return list of closing prices, most recent last

**Usage**: Called by pre-market service and monitoring checkpoints. RiskEngine computes volatility regime from these closes.

**`fetch_sensex() -> float`**:
1. Fetch current Sensex level from AlphaVantage intraday endpoint (latest bar close)
2. Used at CP1 and monitoring checkpoints for circuit breaker check
3. Return current Sensex value

---

## 4. Per-Module Logging

### 4.1 Architecture

**4 log files** (per US-9.4):
| Logger Name | File | Modules Routed |
|---|---|---|
| `at.data` | `logs/data.log` | DataPipeline, Bhavcopy/AlphaVantage/KotakNeo adapters |
| `at.model` | `logs/model.log` | FeatureEngine, ModelManager, PortfolioOptimizer, BacktestEngine |
| `at.execution` | `logs/execution.log` | RiskEngine, ExecutionEngine, WatchlistManager |
| `at.scheduler` | `logs/scheduler.log` | Scheduler, all Service classes, StartupService, ShutdownService |

**Root logger**: `at` — catches any uncategorized log output.

### 4.2 Configuration

Log levels per module set in `config/config.yaml` → `logging.levels` section.

**`setup_logging(config: dict) -> None`** (called once at startup):
1. Create `logs/` directory if not exists
2. For each logger (data, model, execution, scheduler):
   a. Create `TimedRotatingFileHandler` with `when='midnight'`, `backupCount=30`
   b. Set format: `[%(asctime)s] [%(name)s] [%(levelname)s] %(message)s` with `datefmt='%Y-%m-%d %H:%M:%S'`
   c. Set level from config (default: INFO)
3. Add console handler (stdout) for the root `at` logger at WARNING level (for systemd journal)
4. Log "Logging initialized" at INFO to each logger as a startup canary

### 4.3 Log Format

**Structured format**: `[YYYY-MM-DD HH:MM:SS] [at.module] [LEVEL] message`

**Examples**:
```
[2026-03-05 16:00:05] [at.data] [INFO] EOD fetch complete: 42/45 symbols validated, 3 rejected
[2026-03-05 16:00:05] [at.data] [WARNING] Symbol XYZLTD rejected: close=0.0 (zero price)
[2026-03-05 09:30:12] [at.execution] [INFO] G-006 trailing stop triggered: RELIANCE at ₹2340 (5.1% below peak ₹2466)
[2026-03-05 09:30:12] [at.scheduler] [INFO] CP1 started
[2026-03-05 09:31:55] [at.scheduler] [INFO] CP1 completed in 103.2s
```

### 4.4 Rotation & Retention

- **Rotation**: Daily at midnight IST (`TimedRotatingFileHandler` with `when='midnight'`)
- **Retention**: 30 days (`backupCount=30`)
- **Compression**: Post-rotation compression via external `logrotate` config (compress files older than 2 days)
- **Disk alert**: If total `logs/` directory exceeds threshold or NVMe >80%, log WARNING + Telegram alert

### 4.5 Security

- **Never log credentials**: API keys, tokens, passwords must never appear in log output
- **Scrub before logging**: Any message containing configurable sensitive patterns (e.g., "token", "api_key", "password") triggers a warning if detected in log output during testing
- **File permissions**: Log files created with owner-only read/write (0o600 on Linux)
