# NFR Requirements — Unit 1: Foundation & Data

> **Scope**: ConfigManager, StateManager, DataPipeline, Per-Module Logging  
> **Source**: Requirements_Formal.md §10.1, §11, §13 + Functional Design Q&A answers

---

## 1. Performance Requirements

### P-001: Memory Budget — Unit 1 Components
| Component | Budget | Context |
|---|---|---|
| Qlib data pipeline (50 stocks, 2yr) | 1.0–1.5 GB | Resident during evening batch and checkpoint |
| Feature matrix in memory | ~0.5 GB (shared with Unit 2) | Loaded by DataPipeline, consumed by FeatureEngine |
| SQLite + StateManager | < 50 MB | File-based, minimal RAM (OS page cache handles hot pages) |
| ConfigManager | < 10 MB | Parsed YAML in memory — negligible |
| Logging infrastructure | < 10 MB | File handlers with memory-mapped buffers |
| **Unit 1 total** | **~1.5–2.0 GB** | Fits comfortably within 8 GB budget |

### P-002: AlphaVantage Rate Limit Compliance
- **Constraint**: Max 5 API calls/minute (free tier)
- **Implementation**: Minimum 12-second spacing between consecutive calls
- **Bootstrap impact**: 51 calls × 12s = ~10.2 minutes (one-time, acceptable)
- **Checkpoint impact**: 10-15 calls × 12s = 2-3 minutes (within 10-min timeout)

### P-003: Checkpoint Time Budget — DataPipeline Share
- **Total checkpoint timeout**: 10 minutes (E-012)
- **DataPipeline allotment**: ≤ 3 minutes for intraday price fetch
- **Rationale**: Remaining 7 minutes for inference (Unit 2) + risk checks (Unit 3) + order submission (Unit 3)
- **Achieved by**: Limiting intraday fetches to held + proposed symbols only (~10-15 calls)

### P-004: Evening Batch — DataPipeline Share
- **Bhavcopy download + parse**: < 30 seconds (2-3 MB CSV, local filter)
- **Sensex fetch**: 1 AlphaVantage call, < 15 seconds
- **Qlib append**: < 30 seconds (incremental, not full rewrite)
- **Total DataPipeline time**: < 2 minutes of the evening batch window

### P-005: SQLite Write Performance
- **WAL mode**: Enabled for concurrent read during checkpoint while evening batch writes
- **Expected write volume**: < 100 rows/day across all tables (orders, audit, checkpoint log)
- **No write bottleneck**: SQLite handles thousands of writes/second; our load is trivial

### P-006: Bootstrap Completion Time
- **Target**: < 15 minutes for 51 symbols from AlphaVantage
- **Blocking**: System startup blocked until complete (SP-001)
- **One-time only**: Not repeated after initial run

---

## 2. Reliability & Recovery Requirements

### R-001: SQLite Data Integrity
- **WAL checkpoint**: Run `PRAGMA wal_checkpoint(TRUNCATE)` after each evening batch
- **Integrity checks**: `PRAGMA integrity_check` on every startup (E-011)
- **Foreign keys**: Always enabled (`PRAGMA foreign_keys = ON`)
- **Atomic transactions**: All multi-row operations wrapped in transactions

### R-002: Crash Recovery — State Completeness
- **All critical state persisted to SQLite**: Portfolio holdings, order history, drawdown state, checkpoint log, system state flags
- **Recovery detection**: `StateManager.detect_recovery_state()` reconstructs system state from SQLite on startup
- **Checkpoint idempotency**: SP-007 prevents duplicate order execution after crash
- **Zero data loss guarantee**: SQLite WAL + fsync ensures no committed data is lost

### R-003: Data Source Resilience
- **AlphaVantage**: 3 retries with exponential backoff (12s, 24s, 48s). On permanent failure, skip symbol, continue with others.
- **BSE Bhavcopy**: 3 retries at 15-minute intervals. On failure, evening batch proceeds without new EOD data; plan flagged stale.
- **Telegram**: Queue locally, retry every 5 minutes until reachable (E-009).

### R-004: Bootstrap Resilience
- **Partial failure tolerance**: If ≤20% of symbols fail bootstrap, system proceeds with warning
- **Retry capability**: Failed symbols can be retried independently
- **Idempotent**: Re-running bootstrap skips already-completed symbols

### R-005: Configuration Resilience
- **Fail-fast**: Invalid config prevents startup entirely (not partial startup)
- **All validation at load time**: No runtime config parse errors possible
- **Credential availability**: Missing credentials block startup with clear error message

---

## 3. Security Requirements

### S-001: Credential Protection — Phase 1
- **Storage**: `config/secrets.yaml` excluded from git (`.gitignore`)
- **In-memory**: Cached after first read, never logged
- **File permissions**: 0o600 (owner read/write only) on Linux VPS
- **Scope**: AlphaVantage API key, Telegram bot token, Telegram chat ID
- **Phase 2 upgrade path**: Fernet symmetric encryption with machine-derived key

### S-002: Log Scrubbing (LR-001)
- **Rule**: No credential values in log output — ever
- **Implementation**: Log credential access by name only ("Accessed: alphavantage_api_key"), never value
- **Testing**: Integration test scans log output for sensitive patterns

### S-003: SQLite File Protection
- **File permissions**: 0o600 on `data/autotrader.db` (Linux)
- **No remote access**: SQLite is local file — no network exposure
- **WAL file**: Same permissions as main database file

### S-004: Config File Protection
- **`config.yaml`**: May contain non-sensitive defaults (OK to commit)
- **`secrets.yaml`**: MUST be in `.gitignore`, MUST NOT be committed
- **`.gitignore` validation**: Startup check confirms `secrets.yaml` is not tracked

### S-005: Input Validation at System Boundary
- **BSE Bhavcopy CSV**: All fields validated before use (DV-001 through DV-004)
- **AlphaVantage JSON**: Response structure validated, error responses handled (DS-002)
- **Config YAML**: Schema validated at load time (CR-001 through CR-007)
- **No SQL injection risk**: All SQLite access via parameterized queries through StateManager

---

## 4. Observability Requirements

### O-001: Per-Module Logging
- **4 log files**: `data.log`, `model.log`, `execution.log`, `scheduler.log`
- **Structured format**: `[YYYY-MM-DD HH:MM:SS] [at.module] [LEVEL] message`
- **Configurable levels**: Per-module via `config.yaml` logging section
- **Independent tailing**: Admin can `tail -f logs/data.log` without unrelated noise

### O-002: Log Rotation & Retention
- **Rotation**: Daily at midnight IST (`TimedRotatingFileHandler`)
- **Retention**: 30 days (`backupCount=30`)
- **Compression**: Post-2-day files compressed via system logrotate
- **Disk alert**: Telegram notification if NVMe > 80% usage

### O-003: Startup Diagnostics (E-011)
8 diagnostic checks before normal operation:
1. Database integrity (`PRAGMA integrity_check`)
2. Schema version match
3. Bootstrap status
4. Model artifact existence (post-Unit 2)
5. Config file parseable
6. Credentials present
7. Disk space < 80%
8. Log files writable

### O-004: Performance Logging
- **Operations > 5 seconds**: Logged at INFO with duration
- **Checkpoint lifecycle**: Start/end/duration/status for every checkpoint
- **API call timing**: Each AlphaVantage/Bhavcopy call logged with duration

---

## 5. Maintainability Requirements

### M-001: Schema Versioning
- **Version stored in**: `system_state.schema_version`
- **No auto-migration**: Version mismatch blocks startup, manual intervention required
- **Rationale**: Phase 1 system with single operator — auto-migration adds risk without benefit

### M-002: Configuration as Documentation
- **YAML with comments**: Self-documenting config file
- **All guardrails in config**: No magic numbers in code — all thresholds configurable
- **Change logging**: Every guardrail change recorded in `config_changes` table (G-012)

### M-003: BSE Calendar Maintenance
- **Annual update**: `bse_holidays.yaml` must be updated before each new year
- **Validation at startup**: Missing calendar year blocks trading flows
- **Alert**: Telegram notification if current year holidays not loaded

### M-004: Backup Strategy
- **Daily backup**: After evening batch (~6:15 PM IST) to Hetzner Storage Box
- **Scope**: SQLite DB, model artifact, config files
- **Retention**: 30 days
- **Not Unit 1 responsibility**: BackupService is in Unit 4, but StateManager must support DB file copying safely (via SQLite backup API or WAL checkpoint before copy)

---

## 6. Scalability Requirements

### SC-001: Watchlist Growth
- **Current max**: 50 stocks (G-011)
- **Data pipeline scales linearly**: 50 stocks is well within AlphaVantage and Bhavcopy capacity
- **No action needed**: System is designed for 50 stocks max, no horizontal scaling required

### SC-002: Historical Data Growth
- **Growth rate**: ~50 stocks × 1 row/day = 50 rows/day in Qlib + SQLite
- **2-year window**: ~25,000 rows total per stock, ~1.25M rows for 50 stocks
- **No pruning needed**: 160 GB NVMe has abundant capacity; SQLite handles millions of rows efficiently
- **Qlib binary format**: Compact, grows incrementally

### SC-003: SQLite Capacity
- **Expected DB size**: < 100 MB after years of operation
- **Transaction volume**: < 100 writes/day
- **Read volume**: < 1000 reads/day
- **Conclusion**: SQLite is massively overprovisioned for this workload — no scaling concern

---

## 7. Availability Requirements

### A-001: Uptime Target
- **Trading hours** (9:15 AM – 3:30 PM IST): Target 99.5% availability (allows ~1 missed checkpoint/month)
- **Non-trading hours**: No availability requirement beyond daily batch completion
- **Weekend**: System runs heartbeat only — extended downtime acceptable for maintenance

### A-002: Recovery Time Objective (RTO)
- **Crash recovery**: < 2 minutes (SQLite state reload + self-diagnostic)
- **VPS reboot**: < 5 minutes (systemd auto-start)
- **Bootstrap rerun**: ~15 minutes (only if data corruption)

### A-003: Recovery Point Objective (RPO)
- **Zero data loss**: SQLite WAL mode ensures all committed transactions survive crashes
- **Evening batch**: If missed, next-day CP1 runs on stale plan (reduced functionality, not data loss)

---

## 8. Testing NFR Requirements

### T-001: Test Data Isolation
- **Per-test SQLite databases**: Each test creates its own `test_*.db`, destroyed after test
- **No cross-test state contamination**: Fresh fixtures for every test
- **Mock API responses**: Deterministic fixtures for AlphaVantage and Bhavcopy (no real API calls)

### T-002: Fixture Management
- **Setup**: Create test DB → seed known state (holdings, orders, config)
- **Execute**: Run test logic
- **Assert**: Verify expected state
- **Teardown**: Delete test DB (automatic via pytest fixtures or context managers)
- **Retry on failure**: Fix and re-run — clean fixture guarantees reproducibility

### T-003: Boundary Testing
- **Guardrail thresholds**: Test at exact boundary values (e.g., drawdown at 9.99%, 10.0%, 10.01%)
- **Rate limits**: Test at 5 calls/minute boundary
- **Validation rules**: Test edge cases (zero prices, negative volumes, extreme deviations at exactly 20%)

### T-004: Performance Testing
- **Bootstrap timing**: Measure against 15-minute target with mock API (simulated 12s delays)
- **Checkpoint timing**: Measure DataPipeline fetch time against 3-minute budget
- **SQLite write throughput**: Verify < 1 second for typical daily write volume
