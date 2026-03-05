# Unit of Work Dependencies — Auto-Trader (AT)

> **Build order**: Unit 1 → Unit 2 → Unit 3 → Unit 4 (strictly sequential)

---

## Dependency Matrix

| | Unit 1: Foundation & Data | Unit 2: Intelligence | Unit 3: Trading & Communication | Unit 4: Orchestration & Deployment |
|---|---|---|---|---|
| **Unit 1** | — | · | · | · |
| **Unit 2** | **DEPENDS** | — | · | · |
| **Unit 3** | **DEPENDS** | **DEPENDS** | — | · |
| **Unit 4** | **DEPENDS** | **DEPENDS** | **DEPENDS** | — |

The dependency graph is a simple linear chain — no cycles, no optional dependencies.

---

## Detailed Dependencies

### Unit 1: Foundation & Data → (no upstream dependencies)

**Produces for downstream units**:
- ConfigManager instance (used by all components)
- StateManager instance (used by all components that persist state)
- DataPipeline instance (used by FeatureEngine, RiskEngine, ExecutionEngine, WatchlistManager)
- Per-module logging infrastructure (used by all components)
- SQLite schema and CRUD operations
- Qlib dataset with validated historical data

**External dependencies**: BSE Bhavcopy servers, AlphaVantage API, SQLite, filesystem

### Unit 2: Intelligence → Unit 1

**Requires from Unit 1**:
- DataPipeline: Clean OHLCV data in Qlib format
- StateManager: Model artifact persistence, portfolio state (for cold-start day)
- ConfigManager: Model hyperparameters, prediction horizon, top-N count

**Produces for downstream units**:
- FeatureEngine: Feature matrices for RiskEngine (via BacktestEngine) 
- ModelManager: Alpha scores for PortfolioOptimizer → RiskEngine pipeline
- PortfolioOptimizer: Trade proposals for RiskEngine
- BacktestEngine: Viability assessment results for ReportGenerator

**External dependencies**: LightGBM library, Qlib MVO library

### Unit 3: Trading & Communication → Units 1, 2

**Requires from Unit 1**:
- DataPipeline: Intraday prices, Sensex level + history (for RiskEngine checks)
- StateManager: Holdings, drawdown state, highest-seen prices, order log
- ConfigManager: Guardrail thresholds, operating mode, API credentials, bot config

**Requires from Unit 2**:
- PortfolioOptimizer: Trade proposals (input to RiskEngine)
- PortfolioOptimizer: Top-N history (input to WatchlistManager screener)
- ModelManager: Model info (input to ReportGenerator)
- BacktestEngine: Results (input to ReportGenerator)

**Produces for downstream units**:
- All domain components complete — services can orchestrate the full pipeline

**External dependencies**: OpenAlgo (localhost HTTP), Telegram Bot API (HTTPS)

### Unit 4: Orchestration & Deployment → Units 1, 2, 3

**Requires from Units 1-3**: All 12 domain components must be functional. Services call component methods directly — they need the full API surface.

**Produces**: The running system — all scheduled jobs, deployment configuration, test suite.

**External dependencies**: cron (Linux), systemd, Hetzner Storage Box (SSH/rsync), VPS infrastructure

---

## Build Order Visualization

```
Unit 1: Foundation & Data
  ConfigManager ──┐
  StateManager  ──┼── leaf nodes (no dependencies)
  Logging       ──┘
  DataPipeline  ──── depends on CM + SM
        │
        ▼
Unit 2: Intelligence
  FeatureEngine ──── depends on DataPipeline
  ModelManager  ──── depends on FeatureEngine + SM + CM
  PortfolioOpt  ──── depends on ModelManager + SM + CM
  BacktestEngine ─── depends on FE + MM + RE (forward ref resolved at integration)
        │
        ▼
Unit 3: Trading & Communication
  RiskEngine      ──── depends on DP + PO + TN + SM + CM
  ExecutionEngine ──── depends on DP + RE + TN + CM
  WatchlistMgr    ──── depends on DP + PO + TN + SM + CM
  TelegramNotifier ─── depends on RG + CM
  ReportGenerator  ─── depends on MM + RE + EE + SM + BE
        │
        ▼
Unit 4: Orchestration & Deployment
  All Services    ──── depend on all components from Units 1-3
  systemd + cron  ──── infrastructure layer
  Test suite      ──── validates all units end-to-end
```

---

## Integration Points Between Units

| From → To | Integration Point | Data Exchanged |
|---|---|---|
| Unit 1 → Unit 2 | DataPipeline → FeatureEngine | Qlib-structured OHLCV data |
| Unit 2 → Unit 2 | FeatureEngine → ModelManager | Feature matrix |
| Unit 2 → Unit 2 | ModelManager → PortfolioOptimizer | Alpha scores per stock |
| Unit 2 → Unit 3 | PortfolioOptimizer → RiskEngine | Trade proposals |
| Unit 2 → Unit 3 | PortfolioOptimizer → WatchlistManager | Top-N history |
| Unit 2 → Unit 3 | ModelManager → ReportGenerator | Model info + metrics |
| Unit 1 → Unit 3 | DataPipeline → RiskEngine | Sensex (level + history), stock prices |
| Unit 3 → Unit 3 | RiskEngine → ExecutionEngine | Approved proposals |
| Unit 3 → Unit 3 | ReportGenerator → TelegramNotifier | Formatted reports |
| Units 1-3 → Unit 4 | All components → Services | Full method API surface |

### BacktestEngine Forward Reference

BacktestEngine (Unit 2) has a dependency on RiskEngine (Unit 3) for guardrail evaluation during simulation. **Resolution**: During Unit 2 development, BacktestEngine will use a simplified stub for guardrail evaluation. The real RiskEngine integration is completed when Unit 3 is built. This is the only cross-unit forward reference.
