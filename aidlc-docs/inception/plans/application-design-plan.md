# Application Design Plan — Auto-Trader (AT)

> **Stage**: Application Design (INCEPTION)  
> **Depth**: Standard  
> **Date**: 2025-03-05

---

## Design Assessment

The formal requirements (638 lines, v3.7), 12 verification Q&A, and 31 user stories provide sufficient clarity to identify all system components and their boundaries without additional questions. Key design decisions the user deferred to AI (Q8 project structure, Q9 backtest thresholds, Q10 feature set) will be resolved in the artifacts below.

## Component Architecture Plan

### Phase 1: Component Identification
- [x] Identify all major functional components from requirements and stories
- [x] Define component boundaries and single-responsibility assignments
- [x] Map components to requirement sections and user stories

### Phase 2: Mandatory Artifact Generation

- [x] Generate `components.md` with component definitions and high-level responsibilities
  - [x] DataPipeline — Data ingestion, validation, Qlib dataset management
  - [x] FeatureEngine — Technical feature computation, feature matrix
  - [x] ModelManager — LightGBM training, inference, artifact management
  - [x] PortfolioOptimizer — Stock ranking, Qlib MVO, position weight calculation
  - [x] RiskEngine — All guardrails (G-001 to G-013)
  - [x] ExecutionEngine — OpenAlgo integration, order lifecycle
  - [x] WatchlistManager — Watchlist CRUD, Saturday screener, auto-removal
  - [x] TelegramNotifier — Bot messaging, HITL approval workflow
  - [x] ReportGenerator — Daily reports, audit trails, backtest reports
  - [x] StateManager — SQLite persistence, crash recovery
  - [x] ConfigManager — Configuration, mode management, guardrail config
  - [x] BacktestEngine — Historical backtesting framework

- [x] Generate `component-methods.md` with method signatures (I/O types)
  - [x] DataPipeline methods (fetch, validate, bootstrap, calendar)
  - [x] FeatureEngine methods (compute features, build matrix)
  - [x] ModelManager methods (train, predict, load/save artifacts)
  - [x] PortfolioOptimizer methods (rank, optimize, generate actions)
  - [x] RiskEngine methods (evaluate guardrails, position sizing, drawdown)
  - [x] ExecutionEngine methods (submit orders, cancel, track fills)
  - [x] WatchlistManager methods (seed, add/remove, screen, auto-remove)
  - [x] TelegramNotifier methods (send, receive, HITL approval)
  - [x] ReportGenerator methods (daily report, audit entry, backtest report)
  - [x] StateManager methods (save/load portfolio, checkpoint state)
  - [x] ConfigManager methods (load config, get mode, update guardrail)
  - [x] BacktestEngine methods (run backtest, evaluate, report)

- [x] Generate `services.md` with service definitions and orchestration patterns
  - [x] EveningBatchService — Post-close pipeline orchestration
  - [x] CheckpointService — Intraday CP1/CP2/CP3 orchestration
  - [x] WeeklyService — Saturday screener + Sunday retrain
  - [x] StartupService — Bootstrap, self-diagnostic, recovery
  - [x] ShutdownService — Graceful shutdown
  - [x] HeartbeatService — Trading day + weekend heartbeats
  - [x] BackupService — Daily backup to Hetzner Storage Box
  - [x] Scheduler — Top-level cron orchestration

- [x] Generate `component-dependency.md` with dependency matrix and data flow
  - [x] Dependency matrix (component × component)
  - [x] Data flow: Evening Batch pipeline
  - [x] Data flow: CP1 (full decision) pipeline
  - [x] Data flow: CP2/CP3 (monitoring) pipeline
  - [x] Communication patterns

- [x] Validate design completeness and consistency

### Phase 3: Approval
- [x] Log approval prompt in audit.md
- [x] Present completion message with review instructions
- [ ] Await user approval

## Design Decisions Made (AI per Q8/Q10)

| # | Decision | Resolution |
|---|---|---|
| Q8 | Project structure | 12 domain components + 7 orchestration services (see components.md) |
| Qlib split | Where Qlib fits | Data preparation integrated into DataPipeline; MVO into PortfolioOptimizer |
| Backtest | Separate component? | Yes — BacktestEngine is distinct; it orchestrates the pipeline in simulation mode with cost modelling |
| State | SQLite abstraction | StateManager encapsulates all SQLite operations; components never access DB directly |
| Config | Centralized config | ConfigManager manages all config; guardrail values and mode switching go through it |
