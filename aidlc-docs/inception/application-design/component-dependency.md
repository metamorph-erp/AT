# Component Dependencies — Auto-Trader (AT)

> **Scope**: Dependency relationships, communication patterns, and data flow between all 12 domain components and 7 orchestration services.

---

## 1. Dependency Matrix

Each cell shows whether the **row component** depends on the **column component**.  
`D` = direct dependency, `·` = no dependency.

```
                  DP  FE  MM  PO  RE  EE  WM  TN  RG  SM  CM  BE
DataPipeline      ·   ·   ·   ·   ·   ·   D   ·   ·   D   D   ·
FeatureEngine     D   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·
ModelManager      ·   D   ·   ·   ·   ·   ·   ·   ·   D   D   ·
PortfolioOptimizer·   ·   D   ·   ·   ·   ·   ·   ·   D   D   ·
RiskEngine        D   ·   ·   D   ·   ·   ·   D   ·   D   D   ·
ExecutionEngine   D   ·   ·   ·   D   ·   ·   D   ·   ·   D   ·
WatchlistManager  D   ·   ·   D   ·   ·   ·   D   ·   D   D   ·
TelegramNotifier  ·   ·   ·   ·   ·   ·   ·   ·   D   ·   D   ·
ReportGenerator   ·   ·   D   ·   D   D   ·   ·   ·   D   ·   D
StateManager      ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·
ConfigManager     ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·
BacktestEngine    ·   D   D   ·   D   ·   ·   ·   ·   ·   D   ·
```

**Legend**: DP=DataPipeline, FE=FeatureEngine, MM=ModelManager, PO=PortfolioOptimizer, RE=RiskEngine, EE=ExecutionEngine, WM=WatchlistManager, TN=TelegramNotifier, RG=ReportGenerator, SM=StateManager, CM=ConfigManager, BE=BacktestEngine

### Dependency Summary per Component

| Component | Depends On | Depended On By |
|---|---|---|
| **DataPipeline** | StateManager, ConfigManager | FeatureEngine, RiskEngine, ExecutionEngine, WatchlistManager |
| **FeatureEngine** | DataPipeline | ModelManager, BacktestEngine |
| **ModelManager** | FeatureEngine, StateManager, ConfigManager | PortfolioOptimizer, ReportGenerator, BacktestEngine |
| **PortfolioOptimizer** | ModelManager, StateManager, ConfigManager | RiskEngine, WatchlistManager |
| **RiskEngine** | DataPipeline, PortfolioOptimizer, TelegramNotifier, StateManager, ConfigManager | ExecutionEngine, ReportGenerator, BacktestEngine |
| **ExecutionEngine** | DataPipeline, RiskEngine, TelegramNotifier, ConfigManager | ReportGenerator |
| **WatchlistManager** | PortfolioOptimizer, TelegramNotifier, StateManager, ConfigManager | DataPipeline (via service-mediated symbol exchange) |
| **TelegramNotifier** | ReportGenerator, ConfigManager | RiskEngine, ExecutionEngine, WatchlistManager |
| **ReportGenerator** | ModelManager, RiskEngine, ExecutionEngine, StateManager, BacktestEngine | TelegramNotifier |
| **StateManager** | (none — leaf) | DataPipeline, ModelManager, PortfolioOptimizer, RiskEngine, WatchlistManager, ReportGenerator |
| **ConfigManager** | (none — leaf) | All components |
| **BacktestEngine** | FeatureEngine, ModelManager, RiskEngine, ConfigManager | ReportGenerator |

### Service-Mediated Dependency Resolution

**DataPipeline and WatchlistManager are intentionally decoupled** at component level. The orchestration service builds symbol scopes (watchlist for EOD, held + proposed for intraday) and passes them to `DataPipeline.fetch_*` methods.

**Resolution**: WatchlistManager never calls DataPipeline directly. DataPipeline receives symbol lists as input from services. For watchlist additions, services invoke `DataPipeline.bootstrap_history(new_symbols)` after WatchlistManager approves changes.

---

## 2. Communication Patterns

### Pattern 1: Synchronous Function Calls
All component-to-component communication is **synchronous in-process function calls**. No message queues, no async I/O, no microservices. The system runs as a single Python process per service (one for intraday checkpoints, one for evening batch, shared state via SQLite).

### Pattern 2: Service-Mediated Data Passing
Components do not directly call each other in most cases. Services orchestrate the pipeline by calling Component A, capturing the return value, and passing it as input to Component B. This keeps components loosely coupled.

**Example** (EveningBatchService):
```
raw_data = DataPipeline.fetch_eod_data(watchlist, today)
validated = DataPipeline.validate_data(raw_data)
DataPipeline.load_into_qlib(validated)
features = FeatureEngine.compute_features(qlib_data, watchlist, today)
scores = ModelManager.predict(features)
ranked = PortfolioOptimizer.rank_stocks(scores)
# ... service passes each output to the next component
```

### Pattern 3: Shared State via StateManager
Components read/write persistent state through StateManager (SQLite). No direct file or database access by other components. This provides:
- Single source of truth for portfolio, orders, checkpoints
- Crash recovery from known-good persisted state
- Auditability of all state transitions

### Pattern 4: Event-Style Alerts via TelegramNotifier
Components that need to trigger alerts (RiskEngine, ExecutionEngine) hold a reference to TelegramNotifier and call it directly for alerts. This is an exception to the service-mediated pattern because alerts must fire immediately regardless of pipeline stage.

---

## 3. Data Flow Diagrams

### 3a. Evening Batch Pipeline (Trading Day, 4:00 PM)

```
WatchlistManager                  ConfigManager
     |                                 |
     | get_active_watchlist()          | (config for all)
     v                                 v
DataPipeline ──fetch/validate──> FeatureEngine ──features──> ModelManager
                                                                  |
                                                          alpha_scores
                                                                  |
                                                                  v
                                                        PortfolioOptimizer
                                                                  |
                                                        trade_proposals
                                                                  |
                                                                  v
StateManager <──persist──        RiskEngine ──approved──> (baseline plan)
     |                                |
     |                         guardrail alerts
     |                                |
     v                                v
ReportGenerator ──report──> TelegramNotifier
```

**Flow Summary**:
1. Fetch EOD data for watchlist → validate → load into Qlib
2. Compute features → run model inference → alpha scores
3. Rank stocks → optimise portfolio → generate trade proposals
4. Evaluate guardrails → produce baseline plan (no execution in evening)
5. Persist state → generate daily report → send via Telegram

### 3b. CP1 — Full Decision (Trading Day, 9:30 AM)

```
DataPipeline ──intraday prices──> FeatureEngine ──partial features──> ModelManager
                                                                           |
                                                                     alpha_scores
                                                                           |
                                                                           v
                                                                 PortfolioOptimizer
                                                                           |
                                                                   trade_proposals
                                                                           |
                                                                           v
                                    RiskEngine <──drawdown/stop-loss check
                                         |
                                   approved_proposals
                                         |
                                         v
                              ExecutionEngine ──submit orders──> OpenAlgo
                                         |
                                    order_results
                                         |
                                         v
                                    StateManager
```

**Key Difference from Evening Batch**: CP1 uses incremental features on partial intraday bars, and approved proposals are actually executed via ExecutionEngine (not just persisted as a plan).

### 3c. Hourly Monitoring — Mon-1 to Mon-5 (10:30 / 11:30 / 12:30 / 1:30 / 2:30)

```
DataPipeline ──intraday prices + Sensex──> RiskEngine
                                             |
                                    check: stop-loss, drawdown, vol regime
                                             |
                              ┌──────────────┼──────────────┐
                              v              v              v
                      stop-loss sells   drawdown sells   (no buys)
                              |              |
                              v              v
                        ExecutionEngine ──submit──> OpenAlgo
                              |
                    [Mon-1 only] cancel unfilled CP1 orders
                              |
                              v
                         StateManager
```

**Key Difference from CP1**: No new model inference, no new ranking, no buy proposals. Only monitoring: stop-loss checks, drawdown protection, volatility regime enforcement. Unfilled CP1 order cleanup runs at Mon-1 only. Same flow repeats hourly (5 times) ensuring max 1-hour gap between risk checks.

### 3d. Weekly: Saturday Screener + Sunday Retrain

```
Saturday 10:00 AM:
  PortfolioOptimizer ──top-N history──> WatchlistManager ──screener result──> TelegramNotifier
                                              |                                     |
                                              | (await approval)                    |
                                              v                                     v
                                        apply_changes ──new symbols──> DataPipeline.bootstrap_history()

Sunday 2:00 AM:
  FeatureEngine ──full features──> ModelManager.train() ──new artifact──> ModelManager.save_model()
```

---

## 4. Infrastructure Dependencies

| Component | External System | Protocol | Notes |
|---|---|---|---|
| DataPipeline | BSE Bhavcopy servers | HTTPS | EOD OHLCV download |
| DataPipeline | AlphaVantage API | HTTPS/REST | Historical research data |
| DataPipeline | Kotak Neo API | HTTPS/REST | Intraday prices (Phase 2: live trading) |
| DataPipeline | AlphaVantage API | HTTPS/REST | Historical research data + Sensex index |
| ExecutionEngine | OpenAlgo (localhost) | HTTP/REST | Order submission (mock in Phase 1) |
| TelegramNotifier | Telegram Bot API | HTTPS | Notifications and HITL |
| BackupService | Hetzner Storage Box | SSH/rsync | Daily backups |
| StateManager | SQLite | Local file | All persistence |

---

## 5. Layered Architecture View

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ORCHESTRATION LAYER                         │
│  Scheduler │ EveningBatch │ Checkpoint │ Weekly │ Startup/Shutdown │
├─────────────────────────────────────────────────────────────────────┤
│                         DOMAIN LAYER                               │
│  DataPipeline │ FeatureEngine │ ModelManager │ PortfolioOptimizer  │
│  RiskEngine   │ ExecutionEngine │ WatchlistManager │ BacktestEngine│
├─────────────────────────────────────────────────────────────────────┤
│                      CROSS-CUTTING LAYER                           │
│     TelegramNotifier │ ReportGenerator │ StateManager │ ConfigMgr  │
├─────────────────────────────────────────────────────────────────────┤
│                      INFRASTRUCTURE LAYER                          │
│  SQLite │ BSE Bhavcopy │ AlphaVantage │ Kotak Neo │ OpenAlgo │ TG │
└─────────────────────────────────────────────────────────────────────┘
```

- **Orchestration**: Services sequence domain components; no business logic here
- **Domain**: Core trading logic; components are independent and testable
- **Cross-Cutting**: Shared concerns used by multiple domain components
- **Infrastructure**: External systems and persistence
