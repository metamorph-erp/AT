# NFR Design Patterns — Unit 1: Foundation & Data

> **Scope**: Design patterns implementing NFR requirements for ConfigManager, StateManager, DataPipeline, Per-Module Logging  
> **Sources**: nfr-requirements.md (P-001→T-004), tech-stack-decisions.md, business-logic-model.md

---

## 1. Reliability Patterns

### 1.1 Fail-Fast Startup (R-005, CR-001→CR-008)

**Pattern**: Validate everything at load time, reject invalid state before any work begins.

**Design**:
```
startup_sequence():
    1. Load config.yaml → validate schema (all errors collected, not fail-on-first)
    2. Load secrets.yaml → verify all required credentials present
    3. Open SQLite DB → PRAGMA integrity_check
    4. Verify schema_version matches code expectation
    5. Run 8 diagnostic checks (E-011)
    6. If ANY check fails → log all failures → Telegram alert → exit(1)
    7. Only after ALL pass → enter normal operation
```

**Key rule**: No partial startup. Either the system is fully validated and running, or it refuses to start with a complete error report.

### 1.2 SQLite Transaction Boundaries (R-001, SP-005)

**Pattern**: Explicit transaction scope with consistent commit/rollback behavior.

**Design**:
- **Single-row writes** (upsert_holding, update_order_status): Auto-commit — each write is its own implicit transaction
- **Multi-row writes** (evening batch plan save + derived order inserts): Explicit `BEGIN` / `COMMIT` with `try/except` → `ROLLBACK`
- **Read operations**: No transaction needed — WAL mode provides snapshot isolation for reads

**Connection management**:
```python
class StateManager:
    def __init__(self, db_path: str):
        self._conn = sqlite3.connect(db_path)
        self._conn.execute("PRAGMA journal_mode=WAL")
        self._conn.execute("PRAGMA foreign_keys=ON")
        self._conn.row_factory = sqlite3.Row
```

Single long-lived connection per process. No connection pooling needed — single-threaded Python process.

### 1.3 WAL Checkpoint Management (R-001, P-005)

**Pattern**: Explicit WAL checkpoint after heavy write batches.

**Design**:
- After evening batch completes: `PRAGMA wal_checkpoint(TRUNCATE)` — reclaims WAL file space
- Not called during trading hours — let WAL accumulate for fast writes
- Provides clean state for nightly backup (M-004 — BackupService reads a consistent snapshot)

### 1.4 Crash Recovery Detection (R-002)

**Pattern**: State reconstruction from persistent store, not in-memory state.

**Design**:
```
detect_recovery_state():
    1. Read system_state → bootstrap_complete, current_mode, plan_stale_flag
    2. Query checkpoint_log WHERE date = today → which checkpoints ran
    3. Query orders WHERE date = today AND status = 'SUBMITTED' → pending orders
    4. Read drawdown_state → current tier, cooling status
    5. Return RecoveryState object

    Caller decides:
    - If needs_bootstrap → run bootstrap (blocks)
    - If pending_orders exist → log warning (stale orders from crash)
    - If partial checkpoints → resume from next unexecuted checkpoint
    - If drawdown active → enforce existing tier
```

**Key principle**: Never assume in-memory state is correct after restart. Always re-derive from SQLite.

### 1.5 Checkpoint Idempotency (SP-007)

**Pattern**: "Already executed" guard before any checkpoint action.

**Design**:
```
before_checkpoint(checkpoint_name: str):
    existing = state_mgr.get_completed_checkpoints(today)
    if checkpoint_name in existing and status == 'COMPLETED':
        log.info(f"Skipping {checkpoint_name} — already completed today")
        return SKIP
    # proceed with checkpoint
```

Prevents duplicate order submissions after crash recovery + restart.

---

## 2. Resilience Patterns

### 2.1 Tiered Retry with Backoff (R-003, EH-003)

**Pattern**: Source-specific retry policies with exponential backoff.

**Design — 3 retry adapters**:

| Source | Delays | Max Retries | On Exhaust |
|---|---|---|---|
| Indian Stock Market API | 5s, 10s, 20s | 3 | Skip symbol, log warning, continue with remaining |
| BSE Bhavcopy | 15min, 15min, 15min | 3 | Evening batch proceeds without new EOD data; plan flagged stale |
| Telegram | 5min, 5min, 5min, … | 100 retries (~8 hours practical cap) | Messages queue locally, drain on reconnect |

**Implementation pattern**:
```python
def retry_with_backoff(func, delays: list[float], on_exhaust: Callable):
    for attempt, delay in enumerate(delays):
        try:
            return func()
        except (requests.ConnectionError, requests.Timeout) as e:
            log.warning(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(delay)
    return on_exhaust()
```

Each data adapter wraps its HTTP calls with the appropriate retry config.

### 2.2 Partial Failure Tolerance (R-004)

**Pattern**: Continue with degraded data rather than abort entirely.

**Design for bootstrap**:
```
bootstrap_results = []
failed_symbols = []
for symbol in watchlist:
    result = fetch_with_retry(symbol)
    if result.success:
        bootstrap_results.append(result)
    else:
        failed_symbols.append(symbol)

failure_rate = len(failed_symbols) / len(watchlist)
if failure_rate > 0.20:
    raise BootstrapError(f"Too many failures: {len(failed_symbols)}/{len(watchlist)}")
# else: proceed with warning
log.warning(f"Bootstrap partial: {len(failed_symbols)} symbols failed, continuing")
```

**Design for checkpoint intraday fetch**: Same pattern — skip individual symbol failures, log them, continue with available data.

### 2.3 Stale Plan Handling (DS-004)

**Pattern**: Degrade gracefully when fresh data is unavailable.

**Design**:
- If Bhavcopy fetch fails after retries → set `plan_stale_flag = "true"` in `system_state`
- Tomorrow's CP1 checks this flag → log warning "Running on stale plan" → proceed but include flag in Telegram notification
- User receives alert, can intervene manually if needed

---

## 3. Performance Patterns

### 3.1 Token Bucket Rate Limiter (P-002)

**Pattern**: Timestamp-based rate limiter for API call spacing.

**Design**:
```python
class RateLimiter:
    def __init__(self, min_interval_seconds: float):
        self._min_interval = min_interval_seconds
        self._last_call_time: float = 0.0

    def wait(self):
        elapsed = time.monotonic() - self._last_call_time
        if elapsed < self._min_interval:
            time.sleep(self._min_interval - elapsed)
        self._last_call_time = time.monotonic()
```

- `IndianStockMarketAPIAdapter` holds one `RateLimiter(0.5)` instance
- Called before every API request: `self._rate_limiter.wait()`
- No external dependency, thread-safe not needed (single-threaded process)

### 3.2 Checkpoint Time Budget Enforcement (P-003)

**Pattern**: Timeout wrapper for checkpoint phases.

**Design**:
```python
import signal  # Linux only — Windows uses threading.Timer fallback

class CheckpointBudget:
    def __init__(self, budget_seconds: int):
        self._budget = budget_seconds

    def _timeout_handler(self, signum, frame):
        raise TimeoutError(f"Budget of {self._budget}s exceeded")

    def run_with_budget(self, func, *args) -> tuple[Any, float]:
        start = time.monotonic()
        old_handler = signal.signal(signal.SIGALRM, self._timeout_handler)
        signal.alarm(self._budget)
        try:
            result = func(*args)
            signal.alarm(0)  # Cancel alarm on success
            elapsed = time.monotonic() - start
            return result, elapsed
        except TimeoutError:
            elapsed = time.monotonic() - start
            log.error(f"Budget exceeded: {elapsed:.1f}s > {self._budget}s")
            raise
        finally:
            signal.signal(signal.SIGALRM, old_handler)
```

> **Note**: This is the Linux (production) implementation using `signal.SIGALRM`. On Windows (dev), a `threading.Timer` fallback is used — it sets a flag that the function checks cooperatively, which is less precise but functional for development and testing.

- DataPipeline.fetch_intraday() runs within a 180-second (3 min) budget
- Caller (CheckpointOrchestrator in Unit 3) manages overall 10-minute budget
- Budget overrun → log error with timing → E-012 handling (checkpoint fails, recorded as TIMEOUT)

### 3.3 Lazy Config Freeze (P-001 memory)

**Pattern**: Parse once, store as frozen dataclass.

**Design**:
- Config parsed from YAML once at startup → validated → converted to `AppConfig` dataclass
- `AppConfig` and all sub-configs are frozen (`@dataclass(frozen=True)`) — no accidental mutation
- Credential file read once → cached as dict in memory — never re-read from disk

### 3.4 Incremental Qlib Append (P-004)

**Pattern**: Append-only updates to Qlib binary store.

**Design**:
- Evening batch adds one day of OHLCV per symbol — not full rewrite
- Qlib's `dump_bin` with `append=True` mode
- Bootstrap does full write (one-time) — subsequent days are incremental
- Keeps P-004 target of < 30 seconds for daily Qlib update

---

## 4. Security Patterns

### 4.1 Credential Isolation (S-001, S-004)

**Pattern**: Single access point for credentials with in-memory caching.

**Design**:
```python
class ConfigManager:
    _credentials: dict[str, str] | None = None

    def get_credential(self, name: str) -> str:
        if self._credentials is None:
            path = Path("config/secrets.yaml")
            with open(path) as f:
                self._credentials = yaml.safe_load(f)
        if name not in self._credentials:
            raise KeyError(f"Missing credential: {name}")
        # Log access event but NEVER log value
        logger.info(f"Credential accessed: {name}")
        return self._credentials[name]
```

- No other module reads `secrets.yaml` directly — all go through `ConfigManager.get_credential()`
- Startup validates all credentials exist (CR-003) before any module initializes
- S-001 scope: Telegram bot token and Telegram chat ID (Indian Stock Market API requires no key)

### 4.2 Log Scrubbing Filter (S-002, LR-001)

**Pattern**: Custom `logging.Filter` that intercepts and sanitizes log records.

**Design**:
```python
class CredentialScrubber(logging.Filter):
    def __init__(self, sensitive_values: list[str]):
        super().__init__()
        self._patterns = [re.escape(v) for v in sensitive_values if v]
        self._regex = re.compile("|".join(self._patterns)) if self._patterns else None

    def filter(self, record: logging.LogRecord) -> bool:
        if self._regex and isinstance(record.msg, str):
            record.msg = self._regex.sub("***REDACTED***", record.msg)
        # Also scrub args if they contain strings
        if record.args and self._regex:
            record.args = tuple(
                self._regex.sub("***REDACTED***", str(a)) if isinstance(a, str) else a
                for a in (record.args if isinstance(record.args, tuple) else (record.args,))
            )
        return True  # Always emit the record (after scrubbing)
```

> **Scaling note**: The compiled regex contains one pattern per credential value. Phase 1 has ~3 credentials (Telegram token, chat ID, and any future API keys). If Phase 2 adds broker TOTP secrets, session tokens, etc., the regex may grow to ~10 patterns. Performance impact is negligible up to ~50 patterns — benchmark if credential count exceeds this threshold.

- Installed on all 4 loggers at startup
- Initialized with credential values from `secrets.yaml`
- Scrubs both `record.msg` and `record.args` — catches all log formatting styles
- **Testing**: Integration test asserts no credential substrings appear in log output

### 4.3 Parameterized Queries (S-005)

**Pattern**: All SQL via parameterized placeholders — no string formatting.

**Design rule**: Every StateManager method uses `cursor.execute(sql, params)` with `?` placeholders.

```python
# CORRECT
cursor.execute("SELECT * FROM holdings WHERE symbol = ?", (symbol,))

# NEVER
cursor.execute(f"SELECT * FROM holdings WHERE symbol = '{symbol}'")
```

This is a coding standard enforced in review, not a separate component.

### 4.4 Startup Gitignore Validation (S-004)

**Pattern**: Verify credential files are excluded from version control.

**Design**:
```
def validate_gitignore():
    gitignore_path = Path(".gitignore")
    if not gitignore_path.exists():
        log.warning(".gitignore not found — secrets.yaml may be at risk")
        return
    content = gitignore_path.read_text()
    if "secrets.yaml" not in content:
        log.error("SECURITY: secrets.yaml NOT in .gitignore — credentials at risk!")
        # Could block startup in strict mode — for Phase 1, emit warning
```

Part of E-011 startup diagnostics.

---

## 5. Observability Patterns

### 5.1 Logger Factory (O-001)

**Pattern**: Centralized logger creation with consistent configuration.

**Design**:
```python
def create_logger(
    module_name: str,
    log_dir: Path,
    level: str,
    scrubber: CredentialScrubber,
) -> logging.Logger:
    logger = logging.getLogger(f"at.{module_name}")
    logger.setLevel(getattr(logging, level.upper()))

    handler = logging.FileHandler(
        filename=log_dir / f"{module_name}.log",
        mode="a",  # Append mode — logrotate (copytruncate) owns rotation
    )
    # Rotation is handled by logrotate (copytruncate) — see infrastructure-design.md §3.2
    # Using plain FileHandler avoids double-rotation conflicts with TimedRotatingFileHandler
    formatter = logging.Formatter(
        "[%(asctime)s] [%(name)s] [%(levelname)s] %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    logger.addFilter(scrubber)

    return logger
```

- Called once per module at startup: `create_logger("data", ...)`, `create_logger("model", ...)`, etc.
- Returns scoped logger that writes to its own file with credential scrubbing
- Log level configurable per module via `config.yaml` logging section

### 5.2 Diagnostic Runner (O-003, E-011)

**Pattern**: Collect-all diagnostic framework.

**Design**:
```python
@dataclass
class DiagnosticCheck:
    name: str
    passed: bool
    message: str
    duration_ms: float

def run_diagnostics(config_mgr, state_mgr) -> DiagnosticResult:
    checks: list[DiagnosticCheck] = []
    all_passed = True

    for check_fn in [
        check_db_integrity,
        check_schema_version,
        check_bootstrap_status,
        check_model_artifact,
        check_config_parseable,
        check_credentials_present,
        check_disk_space,
        check_log_writable,
    ]:
        # check_model_artifact: On first Sandbox deployment (before first Sunday retrain),
        # no model artifact exists. This check passes with a WARNING: "No model artifact
        # found — expected on initial deployment before first weekly retrain (M-003).
        # Model will be created at next Sunday 2:00 AM retrain."
        # The check only FAILS if bootstrap_complete is true AND the system has been
        # running for > 7 days without a model artifact.
        start = time.monotonic()
        try:
            passed, msg = check_fn(config_mgr, state_mgr)
        except Exception as e:
            passed, msg = False, str(e)
        elapsed = (time.monotonic() - start) * 1000
        checks.append(DiagnosticCheck(check_fn.__name__, passed, msg, elapsed))
        if not passed:
            all_passed = False

    return DiagnosticResult(passed=all_passed, checks=checks)
```

- Runs **all** checks even if early ones fail — complete diagnostic report
- Each check returns `(passed: bool, message: str)`
- Result logged at startup, sent via Telegram if any check fails

### 5.3 Performance Timing Decorator (O-004)

**Pattern**: Automatic logging for operations exceeding a time threshold.

**Design**:
```python
def log_timing(threshold_seconds: float = 5.0):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            start = time.monotonic()
            result = func(*args, **kwargs)
            elapsed = time.monotonic() - start
            if elapsed >= threshold_seconds:
                logger.info(f"{func.__name__} completed in {elapsed:.1f}s")
            return result
        return wrapper
    return decorator
```

- Applied to DataPipeline methods: `fetch_bhavcopy()`, `fetch_intraday()`, `bootstrap()`
- Applied to StateManager heavy operations: `run_diagnostics()`
- Threshold default 5s — only logs slow operations to avoid noise

### 5.4 Checkpoint Lifecycle Logging

**Pattern**: Structured start/end bracketing for checkpoint tracking.

**Design**:
```
checkpoint_start(name):
    log_id = state_mgr.log_checkpoint_start(today, name)
    logger.info(f"=== Checkpoint {name} STARTED ===")
    return log_id

checkpoint_end(log_id, name, status, error=None):
    elapsed = compute_duration(log_id)
    state_mgr.log_checkpoint_end(log_id, status, elapsed, error)
    logger.info(f"=== Checkpoint {name} {status} ({elapsed:.1f}s) ===")
```

Visual `===` delimiters make checkpoint boundaries scannable in `tail -f` output.

---

## 6. Testability Patterns

### 6.1 Fixture Factory (T-001, T-002)

**Pattern**: Reusable factory functions for test data creation.

**Design**:
```python
# conftest.py — shared across all Unit 1 tests

@pytest.fixture
def db_path(tmp_path) -> Path:
    """Fresh SQLite database for each test."""
    return tmp_path / "test.db"

@pytest.fixture
def state_mgr(db_path) -> StateManager:
    """StateManager with initialized empty schema."""
    mgr = StateManager(str(db_path))
    mgr.initialize_schema()
    return mgr

@pytest.fixture
def config_mgr(tmp_path) -> ConfigManager:
    """ConfigManager with valid test config."""
    config_file = tmp_path / "config.yaml"
    config_file.write_text(TEST_CONFIG_YAML)
    secrets_file = tmp_path / "secrets.yaml"
    secrets_file.write_text(TEST_SECRETS_YAML)
    return ConfigManager(str(config_file), str(secrets_file))

@pytest.fixture
def seeded_state_mgr(state_mgr) -> StateManager:
    """StateManager with realistic test data pre-loaded."""
    state_mgr.upsert_holding("RELIANCE", 10, 2500.0, "2026-02-01", 2600.0)
    state_mgr.upsert_holding("TCS", 5, 3800.0, "2026-02-01", 3900.0)
    state_mgr.set_state("bootstrap_complete", "true")
    state_mgr.set_state("schema_version", "1")
    state_mgr.set_state("current_mode", "sandbox")
    return state_mgr
```

**Lifecycle**: `tmp_path` is pytest-provided — automatically created before test, deleted after. No manual cleanup needed.

### 6.2 Mock Data Adapters (T-001)

**Pattern**: `responses` library fixtures for deterministic API mocking.

**Design**:
```python
@pytest.fixture
def mock_indian_stock_api():
    """Mock Indian Stock Market API with realistic response fixtures."""
    with responses.RequestsMock() as rsps:
        rsps.add(
            responses.GET,
            re.compile(r"https://raw\.githubusercontent\.com/.*"),
            json=INDIAN_STOCK_API_DAILY_FIXTURE,
            status=200,
        )
        yield rsps

@pytest.fixture
def mock_bhavcopy():
    """Mock Bhavcopy download with CSV fixture."""
    with responses.RequestsMock() as rsps:
        rsps.add(
            responses.GET,
            re.compile(r"https://.*bseindia.*"),
            body=BHAVCOPY_CSV_FIXTURE,
            status=200,
        )
        yield rsps
```

**Fixture data files**: `tests/fixtures/indian_stock_api_daily.json`, `tests/fixtures/bhavcopy_sample.csv` — committed to repo.

### 6.3 Boundary Test Helpers (T-003)

**Pattern**: Parameterized tests for threshold boundary validation.

**Design**:
```python
@pytest.mark.parametrize("deviation_pct, expected_valid", [
    (19.99, True),    # Just under 20% threshold (DV-005)
    (20.00, False),   # Exactly at threshold
    (20.01, False),   # Just over threshold
])
def test_deviation_validation(deviation_pct, expected_valid):
    record = make_ohlcv(close=100, prev_close=100 / (1 + deviation_pct / 100))
    result = validate_deviation(record)
    assert result.is_valid == expected_valid
```

Applied to: all guardrail thresholds, rate limits, capital limits, drawdown tiers.

### 6.4 Test Configuration Constants

**Pattern**: Separate test config that mirrors production structure but with test-safe values.

**Design**:
```python
TEST_CONFIG_YAML = """
system:
  mode: sandbox
  capital: 50000
  min_order_value: 0
# ... full config with safe test values
"""

TEST_SECRETS_YAML = """
telegram_bot_token: "test-token-not-real"
telegram_chat_id: "test-chat-not-real"
"""
```

Uses real structure but fake credential values — ensures scrubbing tests can verify patterns.
