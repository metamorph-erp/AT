# Unit of Work — Story Map — Auto-Trader (AT)

> **Coverage**: All 34 Phase 1 stories mapped. Every story assigned to exactly one unit (2 stories split across 2 units).

---

## Story Assignment

| Story ID | Story Title | Unit | Components Involved |
|---|---|---|---|
| **US-1.1** | Historical Data Bootstrap | Unit 1 | DataPipeline, StateManager |
| **US-1.2** | Daily EOD Data Collection | Unit 1 | DataPipeline, StateManager |
| **US-1.3** | Intraday Price Fetch | Unit 1 | DataPipeline |
| **US-1.4** | BSE Trading Calendar | Unit 1 | DataPipeline |
| **US-2.1** | Feature Engineering Pipeline | Unit 2 | FeatureEngine |
| **US-2.2** | LightGBM Training | Unit 2 | ModelManager |
| **US-2.3** | LightGBM Prediction | Unit 2 | ModelManager |
| **US-3.1** | Stock Ranking & Portfolio Optimization | Unit 2 | PortfolioOptimizer |
| **US-3.2** | Trade Execution via OpenAlgo | Unit 3 | ExecutionEngine |
| **US-3.3** | Position Exit Logic | Unit 3 | RiskEngine, ExecutionEngine |
| **US-4.1** | Position Sizing Guardrails | Unit 3 | RiskEngine |
| **US-4.2** | Drawdown Protection | Unit 3 | RiskEngine |
| **US-4.3** | Volatility Regime Filters | Unit 3 | RiskEngine |
| **US-4.4** | Operational Limits & Alerting | Unit 3 | RiskEngine, TelegramNotifier |
| **US-5.1** | Evening Batch Orchestration | Unit 4 | EveningBatchService |
| **US-5.2** | Intraday Checkpoint Orchestration | Unit 4 | CheckpointService, Scheduler |
| **US-5.3** | Weekly Jobs | Unit 4 | WeeklyService, Scheduler |
| **US-6.1** | Seed Watchlist Initialization | Unit 3 | WatchlistManager, DataPipeline |
| **US-6.2** | Saturday Screener | Unit 3 | WatchlistManager, TelegramNotifier |
| **US-6.3** | Auto-Removal of Suspended Stocks | Unit 3 | WatchlistManager |
| **US-7.1** | Daily Telegram Report | Unit 3 | ReportGenerator, TelegramNotifier |
| **US-7.2** | Daily Heartbeat | Unit 3 | TelegramNotifier |
| **US-7.3** | Audit Trail & Decision Traceability | Unit 3 | ReportGenerator, StateManager |
| **US-8.1** | Crash Recovery & State Persistence | Unit 1 + Unit 4 | StateManager (Unit 1: schema + CRUD), StartupService (Unit 4: recovery flow) |
| **US-8.2** | Data Source & API Failure Handling | Unit 4 | Services (error handling in orchestration) |
| **US-8.3** | Checkpoint Timeout | Unit 4 | CheckpointService |
| **US-9.1** | Local Development Environment | Unit 1 + Unit 4 | (Unit 1: core setup), (Unit 4: full local test) |
| **US-9.2** | VPS Deployment & Hardening | Unit 4 | Deploy scripts, systemd |
| **US-9.3** | systemd Service Management | Unit 4 | Deploy scripts, systemd |
| **US-9.4** | Per-Module Logging & Observability | Unit 1 | Logging infrastructure |
| **US-9.5** | Backups | Unit 4 | BackupService |
| **US-10.1** | Backtesting Framework | Unit 2 | BacktestEngine |
| **US-11.1** | Telegram Bot Creation & Integration | Unit 3 | TelegramNotifier |
| **US-12.1** | Unit & Integration Tests | Unit 4 | Test suite |

---

## Stories per Unit Summary

| Unit | Story Count | Stories |
|---|---|---|
| **Unit 1: Foundation & Data** | 7 | US-1.1, US-1.2, US-1.3, US-1.4, US-8.1 (partial), US-9.1 (partial), US-9.4 |
| **Unit 2: Intelligence** | 5 | US-2.1, US-2.2, US-2.3, US-3.1, US-10.1 |
| **Unit 3: Trading & Communication** | 13 | US-3.2, US-3.3, US-4.1, US-4.2, US-4.3, US-4.4, US-6.1, US-6.2, US-6.3, US-7.1, US-7.2, US-7.3, US-11.1 |
| **Unit 4: Orchestration & Deployment** | 11 | US-5.1, US-5.2, US-5.3, US-8.1 (remaining), US-8.2, US-8.3, US-9.1 (remaining), US-9.2, US-9.3, US-9.5, US-12.1 |

**Split stories**: US-8.1 and US-9.1 span two units. The foundational parts (SQLite schema, persistence CRUD, Python dev setup) are in Unit 1. The orchestration parts (crash recovery startup flow, full local integration test) are in Unit 4.

---

## Coverage Validation

- **Total unique stories**: 34 (matches stories.md)
- **Stories assigned**: 34 ✓ (36 assignments; US-8.1 and US-9.1 split across 2 units each)
- **Unassigned stories**: 0 ✓
- **Phase 2 stories (Epic 13)**: Not included — out of scope for Phase 1 units
