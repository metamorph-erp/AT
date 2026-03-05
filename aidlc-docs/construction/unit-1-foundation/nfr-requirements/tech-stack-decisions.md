# Tech Stack Decisions — Unit 1: Foundation & Data

> **Scope**: Libraries, tools, and version choices for ConfigManager, StateManager, DataPipeline, Per-Module Logging

---

## 1. Core Runtime

| Decision | Choice | Rationale |
|---|---|---|
| **Python version** | 3.10+ | Required by Qlib; f-strings, match statements, type hints |
| **Package manager** | pip + requirements.txt | Simple, no complex dependency graph; Poetry overhead not justified for single-dev project |
| **Virtual environment** | venv | stdlib, cross-platform (Windows dev + Linux prod) |

---

## 2. Configuration

| Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|
| **Config format** | YAML | JSON, TOML, INI | YAML supports comments, readable nested structures (guardrail tables), standard in Python ecosystem (Q1 answer) |
| **YAML parser** | PyYAML (`pyyaml`) | ruamel.yaml, strictyaml | De facto standard, Qlib already depends on it (no new dependency), mature |
| **Config validation** | Custom Python validation | pydantic, cerberus, jsonschema | Config is loaded once at startup; pydantic adds dependency for one-time validation. Simple `validate_config()` function with descriptive errors is sufficient |
| **Credential storage** | Plaintext `secrets.yaml` + `.gitignore` (Phase 1) | env vars, keyring, Fernet | Phase 1 secrets (AV key, Telegram token) are low-risk. Encryption deferred to Phase 2 for Kotak Neo credentials (Q2 answer) |

---

## 3. State Persistence

| Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|
| **Database** | SQLite 3 | PostgreSQL, Redis, flat files | Single-user, single-process, local-only. SQLite is zero-config, cross-platform, handles our write volume (< 100 writes/day) trivially (Q3 answer) |
| **SQLite driver** | `sqlite3` (stdlib) | sqlalchemy, aiosqlite | stdlib — zero dependencies, sufficient for direct SQL. ORM adds abstraction with no benefit for 9 known tables |
| **WAL mode** | Enabled | Default journal | Allows concurrent reads during checkpoint while evening batch writes |
| **JSON serialization** | `json` (stdlib) | msgpack, pickle | For storing lists/dicts in TEXT columns (ranked_stocks, guardrail_results). Human-readable in DB browser |

---

## 4. Data Pipeline

| Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|
| **HTTP client** | `requests` | httpx, urllib3, aiohttp | Synchronous is fine (no concurrent API calls needed — rate limit forces serial). Battle-tested, simple API |
| **CSV parsing** | `pandas` | csv (stdlib), polars | pandas is already a Qlib dependency; `pd.read_csv()` with dtype specification for Bhavcopy parsing |
| **Qlib integration** | Microsoft Qlib (`qlib`) | Custom binary format | Qlib provides DataHandler, DataLoader, feature operators (CSRank, CSZScore). Standard binary format via `dump_bin` (Q9 answer) |
| **Data validation** | Custom Python functions | great_expectations, pandera | 7 validation rules (DV-001 to DV-007) are simple comparisons. Framework overhead not justified |
| **BSE calendar** | Custom YAML + Python `datetime` | exchange_calendars, pandas_market_calendars | BSE holidays are a short list (~15/year). Loading from YAML is trivial. External packages may not have up-to-date BSE calendars |
| **Rate limiter** | Custom token bucket (in-memory) | ratelimit, tenacity | Single-threaded, single API source. A `time.sleep()` + timestamp tracker is sufficient for 5 calls/min |
| **Retry logic** | Custom with exponential backoff | tenacity | 3 retry scenarios with different delays (AV: 12/24/48s, Bhavcopy: 15min, Telegram: 5min). Custom is clearer than configuring tenacity for 3 different policies |

---

## 5. Logging

| Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|
| **Logging library** | `logging` (stdlib) | structlog, loguru | Zero dependencies, `TimedRotatingFileHandler` provides daily rotation + retention built-in. No log aggregation service exists to benefit from structlog JSON. (Q11 answer) |
| **Log rotation** | `TimedRotatingFileHandler` + system `logrotate` | `RotatingFileHandler` | Daily rotation aligns with trading day cycle. `logrotate` handles compression (compress after 2 days) |
| **Log format** | Custom `Formatter` | JSON format | `[YYYY-MM-DD HH:MM:SS] [at.module] [LEVEL] message` — human-readable for `tail -f` debugging |

---

## 6. Testing

| Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|
| **Test framework** | `pytest` | unittest (stdlib) | pytest fixtures perfect for setup/teardown pattern (create DB → seed → test → destroy). Parameterization for boundary testing. De facto Python standard |
| **API mocking** | `responses` (for `requests`) | httpretty, vcrpy, unittest.mock | Declarative mock definitions for AlphaVantage/Bhavcopy HTTP responses. Clean fixture management |
| **Test database** | In-memory SQLite (`:memory:`) + temp files | Docker containers | `:memory:` for fast unit tests, temp files for integration tests that need file-level operations |
| **Coverage** | `pytest-cov` | coverage.py directly | Integrated with pytest runner |

---

## 7. Development & Build

| Decision | Choice | Alternatives Considered | Rationale |
|---|---|---|---|
| **Linter** | `ruff` | flake8, pylint | Fast, replaces flake8 + isort + pyflakes. Single tool |
| **Type checking** | `mypy` (optional) | pyright, pytype | Type hints used for documentation; strict checking optional for Phase 1 |
| **Formatter** | `ruff format` | black | Integrated with ruff linter |

---

## 8. Dependency Summary

### Production Dependencies (requirements.txt)
```
pyyaml>=6.0
requests>=2.28
pandas>=1.5
numpy>=1.23
qlib>=0.9
lightgbm>=3.3
python-telegram-bot>=20.0
```

### Development Dependencies (requirements-dev.txt)
```
pytest>=7.0
pytest-cov>=4.0
responses>=0.23
ruff>=0.1
mypy>=1.0        # optional
```

### Zero-Install (stdlib)
- `sqlite3` — database
- `logging` — log infrastructure
- `json` — serialization
- `datetime` — timestamps, calendars
- `pathlib` — file paths
- `os` — file permissions
- `enum` — type enums
- `dataclasses` — domain entities
- `time` — rate limiting (sleep)

---

## 9. Deployment Compatibility

| Environment | Support |
|---|---|
| **Windows 10/11** (dev) | All dependencies available. SQLite stdlib. YAML/requests/pandas all cross-platform. |
| **Ubuntu 22.04 LTS** (Hetzner VPS) | Native Python 3.10+. All pip packages available. logrotate pre-installed. |
| **Cross-platform path handling** | Use `pathlib.Path` everywhere — no hardcoded `/` or `\` separators |
