# Business Rules — Unit 1: Foundation & Data

> **Scope**: Validation rules, mode-specific behavior, error handling, constraints  
> **Components**: ConfigManager, StateManager, DataPipeline, Per-Module Logging

---

## 1. Configuration Rules

### CR-001: Config File Existence
- **Rule**: System MUST NOT start if `config/config.yaml` is missing or unparseable
- **Action**: Log error, exit with non-zero code
- **Applies to**: System startup

### CR-002: Secrets File Existence
- **Rule**: System MUST NOT start if `config/secrets.yaml` is missing
- **Action**: Log error listing missing credentials, exit
- **Applies to**: System startup

### CR-003: Required Credentials
- **Rule**: All credentials referenced by enabled features must be present
- **Required Phase 1**: `alphavantage_api_key`, `telegram_bot_token`, `telegram_chat_id`
- **Action**: List all missing keys in error, exit

### CR-004: Mode Validity
- **Rule**: `system.mode` must be exactly one of: `sandbox`, `trial`, `production`
- **Action**: Reject config load with descriptive error

### CR-005: Guardrail Consistency
- **Rule**: `g002_max_positions` must be ≥ `portfolio.top_n`
- **Rationale**: System must be able to hold top_n stocks without exceeding position limit
- **Action**: Reject config load

### CR-006: Cold-Start Ramp Validity
- **Rule**: `portfolio.cold_start_ramp` must be a strictly ascending list of floats where last element = 1.0 and all values between 0 (exclusive) and 1.0 (inclusive)
- **Action**: Reject config load

### CR-007: Schedule Time Format
- **Rule**: All time values in `scheduling` section must be valid HH:MM in 24-hour format
- **Rule**: `monitoring_checkpoints` list must be sorted ascending (earliest first) and non-empty
- **Action**: Reject config load

### CR-008: Guardrail Change Logging (G-012)
- **Rule**: Every change to a guardrail value MUST be logged to `config_changes` table before the new value takes effect
- **Record**: field name, old value, new value, timestamp, who ("system" for mode switches, "admin" for manual edits)

---

## 2. State Persistence Rules

### SP-001: Bootstrap Gate
- **Rule**: System MUST NOT enter Sandbox mode or execute any trading flow until `system_state.bootstrap_complete = "true"`
- **Action**: If bootstrap incomplete, run bootstrap before proceeding. Do not start scheduler.
- **Applies to**: System startup

### SP-002: Schema Version Match
- **Rule**: `system_state.schema_version` must match the version expected by running code
- **Action on mismatch**: Log error, refuse to start. Manual migration required.
- **Rationale**: No auto-migration in Phase 1 to avoid data corruption risk

### SP-003: Holdings Uniqueness
- **Rule**: Only one `holdings` row per symbol (enforced by UNIQUE constraint)
- **Action on conflict**: UPSERT — update quantity, avg_entry_price, highest_seen_price

### SP-004: Order Lifecycle Validity
- **Rule**: Order status transitions must follow valid paths:
  - `SUBMITTED` → `FILLED` | `PARTIAL` | `REJECTED` | `CANCELLED`
  - `PARTIAL` → `FILLED` | `CANCELLED`
  - Terminal states (`FILLED`, `REJECTED`, `CANCELLED`) cannot transition further
- **Action**: Reject invalid transitions with error log

### SP-005: Singleton Drawdown State
- **Rule**: `drawdown_state` table always contains exactly one row (id=1)
- **Initialized at**: Database creation with `peak_capital = system.capital`, `daily_start_value = system.capital`, `tier = NONE`

### SP-006: Daily Start Value Reset
- **Rule**: At the start of each trading day (before any checkpoint), `drawdown_state.daily_start_value` MUST be updated to current portfolio value
- **Applies to**: Pre-market preparation (9:10 AM)

### SP-007: Checkpoint Idempotency
- **Rule**: If a checkpoint is already recorded as COMPLETED for today, it MUST NOT be re-executed
- **Purpose**: Prevents duplicate order submission after crash recovery
- **Check**: Query `checkpoint_log WHERE checkpoint_date = today AND checkpoint_name = X AND status = 'COMPLETED'`

### SP-008: System State Key Registry
- **Rule**: Only predefined keys may be written to `system_state` table (see business-logic-model.md §2.2)
- **Action**: Reject unknown keys with warning log
- **Rationale**: Prevents key sprawl and typo-based bugs

### SP-009: Daily Drawdown Halt Reset
- **Rule**: At pre-market preparation (9:10 AM), if `drawdown_state.tier = DAILY_HALT`, automatically reset tier to `NONE`
- **Action**: Reset tier, update daily_start_value to current portfolio value, log reset at INFO level
- **Rationale**: G-004 specifies "resume automatically next trading day" — this rule implements that behavior

### SP-010: Mode Context on Holdings and Orders
- **Rule**: Every new holding and order record MUST include the current system mode (sandbox/trial/production)
- **Action**: Read `system_state.current_mode` and populate `mode` column on insert
- **Rationale**: Enables audit trail reconstruction and mode-transition analysis

---

## 3. Data Validation Rules

### DV-001: EOD Completeness (D-005)
- **Rule**: Every OHLCVRecord must have non-null values for: open, high, low, close, volume, date, symbol
- **Action**: Reject record, log with reason "incomplete record for {symbol}"

### DV-002: Zero Price Rejection (D-005)
- **Rule**: Reject if any of open, high, low, close = 0
- **Action**: Reject record, log "zero price field for {symbol}"

### DV-003: Negative Value Rejection (D-005)
- **Rule**: Reject if any of open, high, low, close, volume < 0
- **Action**: Reject record, log "negative value for {symbol}"

### DV-004: OHLC Consistency (D-005)
- **Rule**: `high` must be ≥ max(open, close) AND `low` must be ≤ min(open, close)
- **Action**: Reject record, log "OHLC inconsistency for {symbol}"

### DV-005: Extreme Daily Move (D-005)
- **Rule**: If |close_today / close_yesterday - 1| > 0.20 → flag for review
- **Action**: Accept record but add warning annotation. Not auto-rejected because BSE permits 20% circuit limits for some stocks.
- **Log**: WARNING level with deviation percentage

### DV-006: Intraday Staleness (D-005a)
- **Rule**: Reject intraday bar if timestamp age > 15 minutes from checkpoint time
- **Action**: Reject, log "stale intraday data for {symbol}: {age_minutes} min old"

### DV-007: Intraday Extreme Move (D-005a)
- **Rule**: If |current_price / prev_close - 1| > 0.20 → reject
- **Action**: Reject, log "extreme intraday deviation for {symbol}: {pct}%"

### DV-008: Batch Rejection Threshold (E-008)
- **Rule**: If >30% of fetched symbols fail validation → skip entire checkpoint
- **Action**: Skip checkpoint, log error with failed symbol list, send Telegram alert
- **Does NOT apply to**: Evening Batch EOD (partial data is still useful for evening batch)

### DV-009: Bootstrap Gap Tolerance
- **Rule**: During bootstrap, if a symbol has >3 consecutive trading day gaps → flag in BootstrapResult but do NOT reject the symbol
- **Action**: Log WARNING, FeatureEngine will forward-fill gaps ≤3 days, leave longer gaps as NaN
- **Rationale**: Some BSE stocks may have trading suspensions in history; excluding them would reduce universe unnecessarily

---

## 4. Data Source Rules

### DS-001: AlphaVantage Rate Limit
- **Rule**: Maximum 5 API calls per minute (free tier)
- **Implementation**: Enforce minimum 12-second spacing between calls
- **On rate limit response**: Wait 60 seconds, retry once. If still rate-limited, fail the operation.
- **Applies to**: All AlphaVantage endpoints (daily, intraday, search)

### DS-002: AlphaVantage Error Handling
- **Rule**: Handle known error responses:
  - `"Error Message"` key in response → invalid symbol or endpoint
  - `"Note"` key in response → rate limit exceeded
  - HTTP 5xx → transient server error, retry with backoff
- **Retry policy**: Max 3 retries with exponential backoff (12s, 24s, 48s)
- **On permanent failure**: Log error, skip symbol, continue with remaining symbols

### DS-003: Bhavcopy Availability
- **Rule**: BSE publishes Bhavcopy after market close (~3:30-4:00 PM IST)
- **Retry**: If download fails at 4:00 PM, retry at 4:15, 4:30, 4:45 PM (3 retries, 15-min spacing)
- **On failure**: Log error, send Telegram alert. Evening batch proceeds without new EOD data (uses yesterday's close as fallback). Set `plan_stale_flag = "true"`.

### DS-004: Bhavcopy CSV Parsing
- **Rule**: Download full CSV (~5000 rows), filter to watchlist symbols
- **On parse error**: Log the malformed row, skip it, continue parsing remaining rows
- **On wrong column count**: Reject entire file, retry download

### DS-005: Intraday Checkpoint Scope (Phase 1)
- **Rule**: At CP1 and monitoring checkpoints, fetch intraday prices only for:
  - (a) Currently held symbols
  - (b) Today's baseline plan proposal symbols
- **Rationale**: AlphaVantage rate limit (5/min) makes fetching all 50 watchlist stocks impractical within 10-minute checkpoint timeout
- **Typical call count**: 10-15 symbols → ~2-3 minutes

### DS-006: Sensex Symbol Resolution
- **Rule**: Use the AlphaVantage symbol for BSE Sensex (exact symbol to be verified during implementation — likely `BSE:SENSEX` or equivalent)
- **Fallback**: If primary symbol fails, try known alternatives
- **Cache**: Daily Sensex close is also persisted in Qlib dataset for volatility regime computation

---

## 5. Qlib Dataset Rules

### QR-001: Standard Binary Format
- **Rule**: All data stored in Qlib standard binary format via `dump_bin.py` conversion
- **Structure**: Per-instrument directories with per-feature `.day.bin` files
- **Rationale**: Enables Qlib DataHandler/DataLoader and built-in feature operators

### QR-002: Calendar Alignment
- **Rule**: All instruments share the same trading calendar (`calendars/day.txt`)
- **Rule**: Data points must be aligned by trading date — no instrument has data for a non-trading day

### QR-003: Gap Forward-Fill
- **Rule**: Gaps ≤3 consecutive trading days → forward-fill with last known close (at DataPipeline level)
- **Rule**: Gaps >3 days → leave as NaN (FeatureEngine responsibility to handle)

### QR-004: Daily Append
- **Rule**: Evening batch appends new day's data to existing binary files (not full rewrite)
- **Rule**: Calendar and instrument files updated incrementally

### QR-005: Minimum History
- **Rule**: Qlib dataset must contain ≥2 years of data before model training is allowed
- **Check**: Count trading days in calendar. Minimum ~490 trading days (245 BSE trading days/year × 2).

---

## 6. BSE Calendar Rules

### BC-001: Holiday Source
- **Rule**: BSE holidays loaded from `config/bse_holidays.yaml` at startup
- **Rule**: File must be updated manually before each calendar year begins

### BC-002: Trading Day Classification
- **Rule**: A day is a trading day if: (a) it is a weekday (Mon-Fri) AND (b) it is not in the BSE holiday list

### BC-003: Holiday File Validation
- **Rule**: At load time, validate:
  - Year field matches expected year
  - All dates are valid ISO dates
  - All dates fall within the specified year
  - No duplicate dates
- **Action on failure**: Log error, refuse to start. Missing calendar = critical failure.

### BC-004: Calendar Year Transition
- **Rule**: If system detects current year has no holidays loaded, send Telegram alert and refuse to execute trading flows
- **Action**: Continue heartbeat and monitoring, but skip all trading decisions

---

## 7. Logging Rules

### LR-001: Credential Scrubbing
- **Rule**: No credential values (API keys, tokens, passwords) may appear in log output
- **Implementation**: Log only credential names ("Accessed credential: alphavantage_api_key"), never values
- **Testing**: Log output scan during integration tests for sensitive patterns

### LR-002: Log Level Default
- **Rule**: If a module's log level is not specified in config, default to INFO
- **Applies to**: Any new module added without explicit config entry

### LR-003: Exception Logging
- **Rule**: All caught exceptions must be logged with full traceback at ERROR level
- **Rule**: Uncaught exceptions are caught by root logger and logged before system exit

### LR-004: Performance Logging
- **Rule**: All operations taking >5 seconds must log duration at INFO level
- **Format**: `[at.module] [INFO] {operation} completed in {duration:.1f}s`
- **Applies to**: API calls, database operations, Qlib data loading, bootstrap

### LR-005: Checkpoint Lifecycle Logging
- **Rule**: Every checkpoint logs: start time, completion time, duration, status
- **Logger**: `at.scheduler`
- **Format**: `CP1 started` / `CP1 completed in 103.2s` / `CP1 FAILED: {reason}`

---

## 8. Error Handling Rules

### EH-001: Startup Failure
- **Rule**: If any diagnostic check fails (E-011), system MUST NOT start
- **Action**: Log all diagnostic results, send Telegram alert (if Telegram is accessible), exit with non-zero code

### EH-002: Database Connection Failure
- **Rule**: If SQLite database cannot be opened or created → critical failure, system exits
- **Retry**: None — SQLite is local file, if it fails it's a filesystem issue

### EH-003: API Retry Policy
- **Rule**: All external API calls use exponential backoff retry:
  - AlphaVantage: 3 retries, 12s/24s/48s delays
  - BSE Bhavcopy: 3 retries, 15-minute delays
  - Telegram: Retry every 5 minutes until successful (E-009 queue)
- **On exhausted retries**: Log failure, continue with degraded functionality where possible

### EH-004: Telegram Unavailability (E-009)
- **Rule**: If Telegram is unreachable for >5 minutes:
  - Queue all pending alerts locally
  - Retry every 5 minutes
  - Auto-execute stop-loss orders (safety-first, even without user notification)
  - Auto-cancel other proposals (non-stop-loss buys)
- **Recovery**: On reconnection, send all queued alerts in chronological order

### EH-005: Partial Bootstrap Failure
- **Rule**: If bootstrap fails for some symbols but succeeds for others:
  - Continue with successful symbols
  - Log failed symbols
  - Do NOT set `bootstrap_complete = true` if >20% of symbols failed
  - Offer retry for failed symbols only

### EH-006: Midnight Rotation Failure
- **Rule**: If log rotation fails (disk full, permission issue):
  - Log to stderr as last resort
  - Send Telegram alert
  - Continue operation — do not halt trading due to logging failure

### EH-007: WAL Mode Checkpoint
- **Rule**: SQLite WAL file must be checkpointed periodically to prevent unbounded growth
- **Implementation**: Run `PRAGMA wal_checkpoint(TRUNCATE)` after each evening batch completes
- **Threshold**: If WAL file exceeds 50 MB, force checkpoint

---

## 9. Watchlist Persistence Rules

### WP-001: Watchlist Change Logging (W-008)
- **Rule**: Every watchlist addition, removal, or status change MUST be logged to `watchlist` table with timestamp, source, and reason
- **Action**: Insert or update watchlist row before the change takes effect

### WP-002: Active Watchlist Limit
- **Rule**: Count of active watchlist entries (status = 'active') must not exceed G-011 (50 stocks)
- **Action**: Reject addition if limit reached, log warning

### WP-003: Frozen Stock Handling (W-007)
- **Rule**: Stocks suspended/delisted >3 days are set to status='frozen' in watchlist table AND frozen=1 in holdings table
- **Action**: Coordinated update across both tables
