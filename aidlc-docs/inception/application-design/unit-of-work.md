# Units of Work — Auto-Trader (AT)

> **Architecture**: Monolith with logical modules — single Python process, single SQLite database, single VPS  
> **Build Order**: Units 1 → 2 → 3 → 4 (each unit depends on prior units being complete and testable)  
> **Per-Unit Loop**: Each unit goes through CONSTRUCTION phase: Functional Design → NFR Requirements → NFR Design → Infrastructure Design → Code Generation

---

## Unit 1: Foundation & Data

**Purpose**: Core infrastructure and data ingestion — everything else builds on this.

**Components**:
| Component | Role in Unit |
|---|---|
| **ConfigManager** | Centralised config loading, mode management, credential access |
| **StateManager** | All SQLite persistence — schema, CRUD, crash recovery, self-diagnostic |
| **DataPipeline** | All data ingestion (Bhavcopy, AlphaVantage, Kotak Neo, Sensex), validation, Qlib dataset management, BSE calendar |
| Per-module logging | Logging infrastructure: 4 log files, configurable levels, structured format |

**Rationale**: ConfigManager and StateManager are leaf nodes (zero dependencies). DataPipeline depends only on StateManager and ConfigManager (plus WatchlistManager for the symbol list, resolved via service-mediated passing). These three components form the foundation every other unit depends on.

**What's testable after this unit**:
- Load/validate configuration
- Create SQLite schema, persist/load state
- Fetch and validate historical and EOD data
- Bootstrap 2-year history into Qlib
- BSE trading calendar (is_trading_day)
- Per-module logging to separate files
- Self-diagnostic checks

**Stories**: US-1.1, US-1.2, US-1.3, US-1.4, US-9.1 (partial — local dev), US-9.4, US-8.1 (partial — state persistence + self-diagnostic)

---

## Unit 2: Intelligence

**Purpose**: ML pipeline — from raw data to trade proposals.

**Components**:
| Component | Role in Unit |
|---|---|
| **FeatureEngine** | Technical feature computation, feature matrix, partial-bar handling |
| **ModelManager** | LightGBM training, inference, artifact versioning, staleness monitoring |
| **PortfolioOptimizer** | Alpha-score ranking, Qlib MVO, trade action generation, cold-start ramp |
| **BacktestEngine** | Full pipeline simulation with cost modelling, viability assessment |

**Rationale**: These four form the "observe → learn → decide" pipeline. FeatureEngine depends on DataPipeline (Unit 1). ModelManager depends on FeatureEngine. PortfolioOptimizer depends on ModelManager. BacktestEngine needs all three to simulate the full strategy. No dependency on Unit 3 or 4 components.

**What's testable after this unit**:
- Compute features from historical data
- Train LightGBM model, run inference, get alpha scores
- Rank stocks, run MVO, generate trade proposals
- Cold-start ramp logic
- Run full backtest against historical data
- Validate strategy viability

**Stories**: US-2.1, US-2.2, US-2.3, US-3.1, US-10.1

---

## Unit 3: Trading & Communication

**Purpose**: Risk enforcement, trade execution, notifications, and watchlist management.

**Components**:
| Component | Role in Unit |
|---|---|
| **RiskEngine** | All guardrails (G-001 to G-013), drawdown tiers, volatility regime, trailing stop-loss |
| **ExecutionEngine** | OpenAlgo mock order submission, order lifecycle, retry queue |
| **WatchlistManager** | Seed init, Saturday screener, auto-removal, frozen holdings |
| **TelegramNotifier** | Bot setup, messaging, HITL approval workflow |
| **ReportGenerator** | Daily reports, audit trail entries, backtest report formatting |

**Rationale**: These components handle "protect → act → communicate". RiskEngine depends on PortfolioOptimizer (Unit 2) and DataPipeline (Unit 1). ExecutionEngine depends on RiskEngine. TelegramNotifier and ReportGenerator are cross-cutting concerns needed by multiple components in this unit. WatchlistManager needs PortfolioOptimizer for top-N history and TelegramNotifier for approvals.

**What's testable after this unit**:
- Evaluate all guardrail rules against trade proposals
- Submit mock orders to OpenAlgo, track fills
- Trailing stop-loss detection
- Drawdown tier escalation (daily halt → Tier 1 → Tier 2 liquidation)
- Volatility regime filtering
- Telegram bot: send messages, receive approvals
- Daily reports formatted and sent
- Full audit trail per trade
- Seed watchlist, run Saturday screener, apply changes
- Frozen holding management

**Stories**: US-3.2, US-3.3, US-4.1, US-4.2, US-4.3, US-4.4, US-6.1, US-6.2, US-6.3, US-7.1, US-7.2, US-7.3, US-11.1

---

## Unit 4: Orchestration & Deployment

**Purpose**: Wire everything together with services, deploy to VPS, and validate with tests.

**Components**:
| Component | Role in Unit |
|---|---|
| **Scheduler** | Top-level cron orchestration, BSE calendar integration |
| **EveningBatchService** | Post-close pipeline orchestration |
| **CheckpointService** | Pre-market, CP1, hourly monitoring (Mon-1 to Mon-5), close |
| **WeeklyService** | Saturday screener + Sunday retrain orchestration |
| **StartupService** | Bootstrap, self-diagnostic, crash recovery |
| **ShutdownService** | Graceful shutdown with state persistence |
| **HeartbeatService** | Trading day + weekend heartbeat messages |
| **BackupService** | Daily backup to Hetzner Storage Box |

**Rationale**: Services are pure orchestration — they sequence component calls from Units 1-3. They must be built last because they depend on all domain components being complete. Deployment and testing are also here because the full system must exist before end-to-end validation.

**What's testable after this unit**:
- Full evening batch pipeline end-to-end
- Full CP1 decision cycle
- Hourly monitoring cycle (Mon-1 through Mon-5)
- Weekly screener + retrain
- System startup with self-diagnostic and crash recovery
- Graceful shutdown
- Heartbeat messaging
- Daily backup
- systemd service configuration
- VPS deployment and hardening
- Unit tests for guardrails and features
- Integration tests for evening batch and checkpoint pipelines

**Stories**: US-5.1, US-5.2, US-5.3, US-8.1 (remaining — crash recovery flow), US-8.2, US-8.3, US-9.1 (remaining), US-9.2, US-9.3, US-9.5, US-12.1

---

## Code Organization (Q8 Resolution)

```
auto-trader/
├── config/
│   ├── config.yaml              # Main configuration (modes, guardrails, API keys ref)
│   ├── bse_holidays.yaml        # BSE trading calendar
│   └── logging.yaml             # Per-module log level configuration
├── src/
│   ├── __init__.py
│   ├── components/
│   │   ├── __init__.py
│   │   ├── config_manager.py    # ConfigManager
│   │   ├── state_manager.py     # StateManager (SQLite)
│   │   ├── data_pipeline.py     # DataPipeline
│   │   ├── feature_engine.py    # FeatureEngine
│   │   ├── model_manager.py     # ModelManager (LightGBM)
│   │   ├── portfolio_optimizer.py  # PortfolioOptimizer (Qlib MVO)
│   │   ├── risk_engine.py       # RiskEngine
│   │   ├── execution_engine.py  # ExecutionEngine (OpenAlgo)
│   │   ├── watchlist_manager.py # WatchlistManager
│   │   ├── telegram_notifier.py # TelegramNotifier
│   │   ├── report_generator.py  # ReportGenerator
│   │   └── backtest_engine.py   # BacktestEngine
│   ├── services/
│   │   ├── __init__.py
│   │   ├── scheduler.py         # Scheduler (top-level cron)
│   │   ├── evening_batch.py     # EveningBatchService
│   │   ├── checkpoint.py        # CheckpointService
│   │   ├── weekly.py            # WeeklyService
│   │   ├── startup.py           # StartupService
│   │   ├── shutdown.py          # ShutdownService
│   │   ├── heartbeat.py         # HeartbeatService
│   │   └── backup.py            # BackupService
│   ├── models/
│   │   ├── __init__.py
│   │   └── types.py             # Shared data types (OHLCVRecord, TradeProposal, etc.)
│   └── logging_setup.py         # Per-module logging configuration
├── tests/
│   ├── unit/
│   │   ├── test_guardrails.py
│   │   ├── test_features.py
│   │   ├── test_risk_engine.py
│   │   └── ...
│   └── integration/
│       ├── test_evening_batch.py
│       ├── test_checkpoint.py
│       └── ...
├── data/                        # Qlib dataset storage, model artifacts
│   ├── qlib/
│   └── models/
├── logs/                        # Per-module log files (data.log, model.log, etc.)
├── deploy/
│   ├── systemd/                 # systemd service files
│   ├── logrotate/               # logrotate configuration
│   └── setup.sh                 # VPS setup script
├── requirements.txt
├── pyproject.toml
└── README.md
```

**Key decisions**:
- Flat component directory (no sub-packages) — 12 files is manageable, avoids import complexity
- Shared types in `models/types.py` — all data classes used across components
- Config files in YAML — human-readable, editable (per requirements)
- `data/` for Qlib datasets and model artifacts — separate from code
- `deploy/` for infrastructure files — systemd units, logrotate, setup script
- `tests/` split into unit + integration — matching US-12.1 requirements
