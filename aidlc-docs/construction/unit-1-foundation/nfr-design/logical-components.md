# Logical Components — Unit 1: Foundation & Data

> **Scope**: Infrastructure-level components that implement NFR patterns for Unit 1  
> **Note**: Unit 1 is a local single-process Python application with no distributed infrastructure (no queues, caches, or service mesh). These components are Python classes/modules, not external services.

---

## Component Inventory

| # | Component | NFR Pattern | Location |
|---|---|---|---|
| 1 | RateLimiter | Performance (P-002) | `src/infrastructure/rate_limiter.py` |
| 2 | RetryHandler | Resilience (R-003) | `src/infrastructure/retry_handler.py` |
| 3 | CredentialScrubber | Security (S-002) | `src/infrastructure/log_scrubber.py` |
| 4 | LoggerFactory | Observability (O-001) | `src/infrastructure/logger_factory.py` |
| 5 | DiagnosticRunner | Observability (O-003) | `src/infrastructure/diagnostics.py` |
| 6 | CheckpointBudget | Performance (P-003) | `src/infrastructure/checkpoint_budget.py` |
| 7 | TestFixtures | Testability (T-001, T-002) | `tests/conftest.py` |

---

## 1. RateLimiter

**Purpose**: Enforce AlphaVantage API rate limit (5 calls/min = 12-second spacing).

**Interface**:
```python
class RateLimiter:
    def __init__(self, min_interval_seconds: float)
    def wait(self) -> None
        """Block until min_interval has elapsed since last call."""
```

**Consumers**: `AlphaVantageAdapter` (DataPipeline)

**Properties**:
- Stateful: tracks last call timestamp via `time.monotonic()`
- No external dependencies
- Single-threaded — no locking needed

---

## 2. RetryHandler

**Purpose**: Reusable retry-with-backoff for all external HTTP calls.

**Interface**:
```python
def retry_with_backoff(
    func: Callable[[], T],
    delays: list[float],
    retryable_exceptions: tuple[type[Exception], ...],
    on_exhaust: Callable[[], T] | None = None,
    logger: logging.Logger | None = None,
) -> T:
    """Execute func with retries. On exhaust, call on_exhaust or re-raise."""
```

**Consumers**: All 3 data adapters (AlphaVantage, Bhavcopy, Telegram), each with their own delay config.

**Configuration presets**:
```python
RETRY_ALPHAVANTAGE = {"delays": [12, 24, 48], "retryable_exceptions": (ConnectionError, Timeout, HTTPError)}
RETRY_BHAVCOPY = {"delays": [900, 900, 900], "retryable_exceptions": (ConnectionError, Timeout, HTTPError)}
RETRY_TELEGRAM = {"delays": [300] * 100, "retryable_exceptions": (ConnectionError, Timeout)}  # Unlimited retries
```

---

## 3. CredentialScrubber

**Purpose**: Prevent credential values from appearing in any log output.

**Interface**:
```python
class CredentialScrubber(logging.Filter):
    def __init__(self, sensitive_values: list[str])
    def filter(self, record: logging.LogRecord) -> bool
        """Scrub sensitive patterns from record.msg and record.args. Always returns True."""
```

**Consumers**: Installed on all 4 loggers via `LoggerFactory`.

**Lifecycle**:
1. Created once at startup after credentials are loaded
2. Passed to `LoggerFactory.create_logger()` for each module
3. Lives for process lifetime

---

## 4. LoggerFactory

**Purpose**: Create consistently configured per-module loggers.

**Interface**:
```python
class LoggerFactory:
    def __init__(self, log_dir: Path, scrubber: CredentialScrubber)

    def create_logger(self, module_name: str, level: str) -> logging.Logger
        """Create scoped logger: at.{module_name} → logs/{module_name}.log"""
```

**Consumers**: Application startup code — creates 4 loggers (data, model, execution, scheduler).

**Responsibilities**:
- Sets `TimedRotatingFileHandler` with daily rotation, 30-day retention
- Applies consistent format: `[YYYY-MM-DD HH:MM:SS] [at.module] [LEVEL] message`
- Installs `CredentialScrubber` filter on each logger
- Reads log level from `config.yaml` per module

---

## 5. DiagnosticRunner

**Purpose**: Execute all 8 startup health checks and report results.

**Interface**:
```python
class DiagnosticRunner:
    def __init__(self, config_mgr: ConfigManager, state_mgr: StateManager)

    def run_all(self) -> DiagnosticResult
        """Execute all diagnostic checks, return aggregate result."""

    # Individual checks (called by run_all)
    def check_db_integrity(self) -> tuple[bool, str]
    def check_schema_version(self) -> tuple[bool, str]
    def check_bootstrap_status(self) -> tuple[bool, str]
    def check_model_artifact(self) -> tuple[bool, str]
    def check_config_parseable(self) -> tuple[bool, str]
    def check_credentials_present(self) -> tuple[bool, str]
    def check_disk_space(self) -> tuple[bool, str]
    def check_log_writable(self) -> tuple[bool, str]
```

**Properties**:
- Runs all checks even if early ones fail (collect-all, not fail-fast)
- Returns `DiagnosticResult` with per-check detail + aggregate boolean
- Each check timed individually for performance monitoring

---

## 6. CheckpointBudget

**Purpose**: Enforce time budgets on checkpoint phases.

**Interface**:
```python
class CheckpointBudget:
    def __init__(self, budget_seconds: int)

    def run_with_budget(self, func: Callable, *args) -> tuple[Any, float]
        """Run func within budget. Returns (result, elapsed_seconds).
           Raises TimeoutError if budget exceeded (on Linux via signal.alarm)."""
```

**Consumers**: CheckpointOrchestrator (Unit 3) wraps DataPipeline.fetch_intraday() with 180s budget.

**Platform note**:
- Linux (production): Uses `signal.SIGALRM` for preemptive timeout
- Windows (dev): Uses `threading.Timer` for cooperative timeout — less precise but functional for dev/test

---

## 7. TestFixtures (tests/conftest.py)

**Purpose**: Shared pytest fixtures for Unit 1 test isolation and data seeding.

**Fixtures provided**:

| Fixture | Creates | Cleanup |
|---|---|---|
| `db_path` | Temp SQLite file path | Auto (tmp_path) |
| `state_mgr` | StateManager with empty schema | Auto |
| `seeded_state_mgr` | StateManager with sample holdings, orders, system state | Auto |
| `config_mgr` | ConfigManager with test config + fake credentials | Auto |
| `mock_alphavantage` | `responses` mock for AV API | Context manager |
| `mock_bhavcopy` | `responses` mock for BSE download | Context manager |
| `mock_telegram` | `responses` mock for Telegram API | Context manager |

**Fixture data directory**: `tests/fixtures/`
- `alphavantage_daily.json` — realistic daily OHLCV response
- `alphavantage_intraday.json` — realistic 5-min bar response
- `bhavcopy_sample.csv` — BSE Bhavcopy CSV with 20 sample stocks
- `test_config.yaml` — valid config with sandbox mode
- `test_secrets.yaml` — fake credential values

---

## Dependency Graph

```
Application Startup
    ├── ConfigManager.load()
    │   └── reads config.yaml + secrets.yaml
    ├── CredentialScrubber(credential_values)
    │   └── receives sensitive values from ConfigManager
    ├── LoggerFactory(log_dir, scrubber)
    │   └── creates 4 loggers, each with scrubber attached
    ├── StateManager(db_path)
    │   └── opens SQLite, enables WAL + FK
    ├── DiagnosticRunner(config_mgr, state_mgr)
    │   └── runs 8 checks, reports result
    └── DataPipeline(config_mgr, state_mgr, logger)
        ├── AlphaVantageAdapter
        │   ├── RateLimiter(12.0)
        │   └── RetryHandler(delays=[12, 24, 48])
        ├── BhavcopyAdapter
        │   └── RetryHandler(delays=[900, 900, 900])
        └── QlibManager
            └── Qlib dump_bin / DataHandler
```

**Initialization order matters**: ConfigManager first (provides credentials for scrubber), then LoggerFactory (provides loggers for all other components), then StateManager (opens DB), then DiagnosticRunner (validates everything), then DataPipeline (uses all of the above).
