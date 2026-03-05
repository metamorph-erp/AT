# Functional Design Plan — Unit 1: Foundation & Data

## Plan Overview
Design detailed business logic for ConfigManager, StateManager, DataPipeline, and Per-Module Logging.

## Execution Steps

- [x] **Step 1**: Design ConfigManager business logic — config schema, mode management, credential access, change logging
- [x] **Step 2**: Design StateManager business logic — SQLite schema (all tables), CRUD operations, crash recovery state, self-diagnostic
- [x] **Step 3**: Design DataPipeline business logic — data source adapters, validation rules, Qlib integration, BSE calendar, bootstrap flow
- [x] **Step 4**: Design Per-Module Logging — log file routing, structured format, level configuration
- [x] **Step 5**: Define domain entities and types — all shared types for Unit 1 (OHLCVRecord, PortfolioState, etc.)
- [x] **Step 6**: Define business rules — validation rules, mode-specific behavior, error handling rules

## Artifacts to Generate
1. `aidlc-docs/construction/unit-1-foundation/functional-design/business-logic-model.md`
2. `aidlc-docs/construction/unit-1-foundation/functional-design/business-rules.md`
3. `aidlc-docs/construction/unit-1-foundation/functional-design/domain-entities.md`

---

## Questions

### ConfigManager — Configuration Schema

**Q1**: The formal spec mentions config stored as JSON or YAML. Do you have a preference, or should I go with YAML (more readable for the guardrail tables and nested structures)?

[Answer]: **YAML** — more readable for nested guardrail tables, mode-specific overrides, and list-based checkpoint schedules. YAML also supports comments, which is valuable for self-documenting config. Standard choice for Python projects (used by Django, Flask, Ansible, etc.). 

---

**Q2**: For API credential encryption — the formal spec says "encrypted on disk using a machine-specific key" (for TOTP in Phase 2). For Phase 1, the only real secrets are the Telegram Bot token and AlphaVantage API key. Options:
- **A)** Simple approach: credentials in a separate `secrets.yaml` file excluded from git (`.gitignore`), plaintext on disk but file-permission protected — encryption deferred to Phase 2 when Kotak Neo credentials arrive
- **B)** Encrypt from day one using Python `cryptography` library with a machine-derived key (hostname + MAC hash)
- **C)** Use environment variables for all secrets (no file storage)

[Answer]: **A** — Phase 1 secrets (Telegram token, AlphaVantage key) are low-risk; a `.gitignore`-excluded `secrets.yaml` with file-permission protection is proportionate. Encryption infrastructure (machine-derived key, Fernet symmetric encryption) should be built in Phase 2 when Kotak Neo broker credentials and TOTP secrets arrive — those carry real financial risk. The ConfigManager interface (`get_api_credentials()`) stays the same either way, so the switch to encryption is transparent to callers. 

---

**Q3**: Checkpoint times in the formal spec say 9:30/11:00/2:00 but the design adopted hourly monitoring (Mon-1 to Mon-5 at 10:30/11:30/12:30/1:30/2:30). The config should hold the **new** hourly schedule. Should the config also retain the ability to switch back to 3-checkpoint mode, or just hardcode the hourly schedule?
- **A)** Config holds a list of monitoring times — fully flexible (can add/remove checkpoints)
- **B)** Just hardcode the 5 hourly monitoring times in config — simpler

[Answer]: **A** — Store as a list of monitoring times in config.yaml (e.g., `monitoring_checkpoints: ["10:30", "11:30", "12:30", "13:30", "14:30"]`). Costs nothing extra to implement — the Scheduler just iterates the list. Gives flexibility to adjust frequency later without code changes (e.g., reduce to 3 during low-volatility periods). CP1 at 9:30 remains a separate fixed entry since it has different behavior (full decision vs monitoring). Also update formal spec for hourly monitoring, it was missed.

---

### StateManager — SQLite Schema

**Q4**: For the SQLite schema design, I plan to create these tables:
1. `holdings` — current portfolio positions
2. `orders` — full order history (every order ever submitted)
3. `baseline_plans` — evening batch plans
4. `checkpoint_log` — checkpoint completion timestamps
5. `audit_trail` — per-trade decision audit (model version, features, guardrail results)
6. `drawdown_state` — daily/total drawdown tracking
7. `config_changes` — guardrail config change log
8. `top_n_snapshots` — membership history for RP-008
9. `system_state` — key-value store for miscellaneous state (bootstrap_complete, last_fetch_timestamp, etc.)

Any tables you want added or removed?

[Answer]: **Keep all 9 tables as proposed.** This covers all persistence requirements from E-005, E-007, E-010, RP-005a, RP-008, G-012, and US-8.1. The `system_state` key-value table is particularly useful as a catch-all for flags like `bootstrap_complete`, `last_successful_retrain`, `plan_stale_flag`, etc. — avoids needing a new table for every piece of miscellaneous state. 

---

**Q5**: For crash recovery, the checkpoint_log tracks which checkpoints ran today. Should we also persist in-flight order state (orders submitted to OpenAlgo but not yet confirmed) separately from the orders table, or is the orders table with a `status=SUBMITTED` flag sufficient?

[Answer]: **`status=SUBMITTED` flag in the orders table is sufficient.** The orders table already tracks full lifecycle (SUBMITTED → FILLED / PARTIAL / REJECTED / CANCELLED). On crash recovery, StateManager queries `WHERE status = 'SUBMITTED' AND date = today` to find unresolved orders. The E-002 persistent retry queue (for broker-unreachable scenarios) can use a separate `pending_retry_queue` column or a lightweight JSON field on the order row. No need for a separate table — it would just duplicate data. 

---

### DataPipeline — Data Sources

**Q6**: The formal spec mentions "Indian Stock Market API (GitHub)" for real-time prices, but in the requirements analysis you confirmed AlphaVantage for historical data and Kotan Neo for real-time (Phase 2). For **Phase 1 intraday prices** (mock mode), which approach:
- **A)** Use AlphaVantage intraday endpoint (they offer 5-min bars for BSE stocks, but with rate limits — 25 calls/day on free tier)
- **B)** Use last EOD close + random jitter as mock prices (pure simulation, no API call)
- **C)** Fetch from a free Indian market data source (e.g., BSE website scraping, Yahoo Finance BSE) — needs research on reliability
- **D)** Just stub out intraday fetch in Phase 1 with configurable mock data from a JSON file (most predictable for testing)

[Answer]: **A** — We want to use real data everywhere, research, prediction, realtime prices, except for actual order submission and result. For the result, we should assume that order got executed immediately. I want to track the "performance and profit" of the system in realtime in phase 1for 1-3 months before we can activate phase 2. And even keep monitoring it hourly, and exit if needed, again at mock level. 

---

**Q7**: BSE Bhavcopy provides EOD data as a daily CSV file downloadable from the BSE website. The download URL follows a date-based pattern. For the 2-year bootstrap:
- **A)** Download daily Bhavcopy CSVs one by one (simple, ~500 files for 2 years)
- **B)** Use BSE's bulk data download if available
- **C)** Use AlphaVantage's `TIME_SERIES_DAILY` endpoint for bootstrap (single API call per stock, but rate-limited)
- **D)** Combination: AlphaVantage for bootstrap (one-time), Bhavcopy for daily ongoing

[Answer]: **D** — AlphaVantage `TIME_SERIES_DAILY` with `outputsize=full` for the one-time bootstrap (returns up to 20 years of daily data in a single call per stock; ~50 stocks × 1 call = 50 calls, well within daily limits even on free tier). Then BSE Bhavcopy for daily ongoing EOD (free, no rate limits, reliable). This gives the cleanest separation: AlphaVantage handles the heavy historical lift once, Bhavcopy handles the lightweight daily append. The DataPipeline abstracts both behind the same `OHLCVRecord` type. 

---

**Q8**: BSE Bhavcopy contains ALL BSE-listed stocks (~5000). We only need the watchlist stocks (≤50). Should the pipeline:
- **A)** Download the full Bhavcopy CSV and filter locally (simple, ~2-3 MB/day)
- **B)** Try to fetch only specific stocks (if BSE provides per-stock API)

[Answer]: **A** — Download full Bhavcopy CSV and filter locally. It's ~2-3 MB/day (trivial on 160 GB NVMe). BSE doesn't offer a reliable per-stock API for Bhavcopy. Filtering a CSV in-memory by symbol is a one-liner in pandas. The full CSV is also useful for the Saturday screener (W-005) which may need broader market data for liquidity screening beyond the current watchlist. 

---

### DataPipeline — Validation & Qlib

**Q9**: For Qlib dataset structure — Qlib expects data in a specific binary format organized by stock symbol. The standard approach is:
- Raw CSV → Qlib `dump_bin.py` conversion → Qlib binary dataset
- Each stock gets its own directory with feature files

This is well-documented in Qlib. Should I design around the standard Qlib data format, or do you have a custom structure preference?

[Answer]: **Standard Qlib binary format.** Use Qlib's `dump_bin.py` conversion pipeline: raw CSV → Qlib binary dataset with per-instrument directories. This gives us free access to Qlib's built-in feature operators (CSRank, CSZScore, rolling windows) and the DataHandler/DataLoader infrastructure. No reason to reinvent this — Qlib's format is battle-tested for exactly this use case. 

---

**Q10**: ~~India VIX~~ **RESOLVED** — NSE dependency dropped entirely. Volatility regime (G-007/G-008) now uses **Sensex 20-day annualized realized volatility** computed from Sensex daily closes already fetched via AlphaVantage (Q12). No external VIX data source needed. Thresholds: >30% halve positions (G-007), >45% halt buys (G-008).

[Answer]: **N/A** — Question resolved. Sensex realized vol replaces India VIX. Computed internally from Sensex close history (no additional data fetch). G-007 threshold: 30% annualized vol. G-008 threshold: 45% annualized vol.

---

### Per-Module Logging

**Q11**: The design specifies 4 log files (data.log, model.log, execution.log, scheduler.log). Python's `logging` module with `RotatingFileHandler` or `TimedRotatingFileHandler` is the standard approach. Any preference on:
- **A)** Python `logging` module (standard library, zero dependencies)
- **B)** `structlog` (structured logging with JSON output — better for log aggregation)
- **C)** `loguru` (simpler API, automatic rotation)

[Answer]: **A** — Python standard `logging` module. Zero external dependencies, battle-tested, and `TimedRotatingFileHandler` gives us the daily rotation + 30-day retention specified in US-9.4. The structured format `[YYYY-MM-DD HH:MM:SS] [MODULE] [LEVEL] message` is easily achieved with a custom Formatter. No log aggregation service exists in this architecture (single VPS, `tail -f` debugging), so structlog's JSON output adds overhead without benefit. loguru is nice but adds a dependency for no gain over stdlib here. 

---

### General / Cross-Cutting

**Q12**: The formal spec mentions Sensex data for relative-strength features. Sensex is a BSE index. Should we fetch it:
- **A)** From the same BSE Bhavcopy (if Sensex is included as an index row)
- **B)** Separately from a specific BSE index endpoint
- **C)** From AlphaVantage (they track BSE Sensex as `BSE:SENSEX` or similar)
- **D)** Defer to Phase 2 — use a fixed benchmark for Phase 1

[Answer]: **C** — AlphaVantage tracks BSE Sensex (`BSE:SENSEX` or equivalent symbol). For the bootstrap, one `TIME_SERIES_DAILY` call gets 20 years of Sensex history. For daily ongoing, one call/day after market close. This aligns with our D-option for Q7 (AlphaVantage for historical, Bhavcopy for daily stocks). BSE Bhavcopy typically does NOT include index values — it's equity-only. BSE does have an index page but no reliable API. AlphaVantage is the cleanest path. 

---
