# Service Definitions — Auto-Trader (AT)

> **Scope**: Orchestration services that coordinate domain components.  
> **Architecture**: Services contain no business logic — they sequence component calls, handle errors, and manage control flow. The **Scheduler** triggers services based on cron schedules and the BSE trading calendar.

---

## Scheduler (Top-Level Orchestrator)

**Purpose**: Trigger services at the correct times based on cron schedules and BSE trading calendar.

**Responsibilities**:
- Manage cron schedule for all timed events
- Consult BSE trading calendar to skip non-trading days
- Trigger the correct service for each scheduled slot
- Ensure no overlapping service executions
- Log all trigger events to scheduler.log

**Schedule** (all times IST):

| Time | Day Type | Service Triggered |
|---|---|---|
| 9:10 AM | Trading day | CheckpointService.run_premarket() |
| 9:15 AM | Trading day | HeartbeatService.send_trading_heartbeat() |
| 9:30 AM | Trading day | CheckpointService.run_cp1() |
| 10:30 AM | Trading day | CheckpointService.run_monitor("mon-1") |
| 11:30 AM | Trading day | CheckpointService.run_monitor("mon-2") |
| 12:00 PM | Weekend/Holiday | HeartbeatService.send_idle_heartbeat() |
| 12:30 PM | Trading day | CheckpointService.run_monitor("mon-3") |
| 1:30 PM | Trading day | CheckpointService.run_monitor("mon-4") |
| 2:30 PM | Trading day | CheckpointService.run_monitor("mon-5") |
| 3:30 PM | Trading day | CheckpointService.run_close() |
| 4:00 PM | Trading day | EveningBatchService.run() |
| ~6:15 PM | Trading day | BackupService.run() |
| Saturday 10:00 AM | Weekly | WeeklyService.run_screener() |
| Sunday 2:00 AM | Weekly | WeeklyService.run_retrain() |

**Component Dependencies**: ConfigManager (schedule config), DataPipeline (trading calendar), StateManager (checkpoint logging)

---

## 1. EveningBatchService

**Purpose**: Orchestrate the post-close pipeline that produces the next-day baseline plan.

**Trigger**: Scheduler at 4:00 PM IST on trading days.

**Orchestration Flow**:
```
1. DataPipeline.fetch_eod_data(watchlist, today)
2. DataPipeline.validate_data(raw_data)
   └─ On failure: skip pipeline, TelegramNotifier.send_alert(), flag plan stale → EXIT
3. DataPipeline.load_into_qlib(validated_data)
4. FeatureEngine.compute_features(qlib_data, watchlist, today)
5. ModelManager.predict(feature_matrix) → alpha_scores
   └─ On anomalous output: discard, TelegramNotifier.send_alert() → EXIT
6. PortfolioOptimizer.rank_stocks(alpha_scores)
7. PortfolioOptimizer.optimize_portfolio(ranked, top_n, holdings, capital)
8. PortfolioOptimizer.generate_trade_actions(optimization, holdings, cold_start_day)
9. RiskEngine.evaluate_proposals(proposals, portfolio_state, market_data)
   └─ Log all guardrail results
10. StateManager.save_baseline_plan(plan)
11. StateManager.save_top_n_snapshot(top_n)
12. StateManager.save_portfolio_state(state)
13. ReportGenerator.generate_daily_report(state, trades, guardrails, model_info)
14. TelegramNotifier.send_report(daily_report)
15. StateManager.save_checkpoint_progress("evening_batch", "complete")
```

**Error Handling**:
- Data source unavailable (E-001): Skip pipeline, send Telegram alert, flag plan stale
- Model anomalous output (M-007): Discard batch, alert, retain previous plan
- Any unhandled exception: Log full traceback, send Telegram alert, persist partial state

**Component Dependencies**: DataPipeline, FeatureEngine, ModelManager, PortfolioOptimizer, RiskEngine, StateManager, ReportGenerator, TelegramNotifier, ConfigManager

---

## 2. CheckpointService

**Purpose**: Orchestrate intraday checkpoints — CP1 (full decision), Mon-1 through Mon-5 (hourly monitoring).

**Trigger**: Scheduler at 9:10 AM (pre-market), 9:30 AM (CP1), then hourly monitoring at 10:30/11:30/12:30/1:30/2:30, and 3:30 PM (close).

### 2a. Pre-Market (9:10 AM)

```
1. DataPipeline.fetch_sensex_history(20)
2. RiskEngine.compute_volatility_regime(sensex_closes) → regime
3. Load baseline plan from StateManager
4. RiskEngine.apply_volatility_adjustment(baseline_proposals, regime)
   └─ If G-008 (vol>45%): filter to sells/exits only
5. StateManager.save_checkpoint_progress("premarket", "complete")
```

No orders submitted. Pre-market only applies regime filter to the baseline plan.

### 2b. CP1 — Full Decision (9:30 AM)

```
1. Build CP1 symbol scope = held symbols + today's baseline proposal symbols
2. DataPipeline.fetch_intraday_prices(cp1_symbols)
3. DataPipeline.fetch_sensex()
4. DataPipeline.validate_data(intraday_bars)
   └─ If >30% fail (E-008): skip checkpoint, log, alert → EXIT
5. FeatureEngine.compute_incremental_features(qlib_data, watchlist, bars)
6. ModelManager.predict(partial_features) → alpha_scores
   └─ On anomalous: discard, use baseline plan
7. ModelManager.is_stale() → if HALT: suppress buys
8. PortfolioOptimizer.rank_stocks(alpha_scores)
9. PortfolioOptimizer.optimize_portfolio(ranked, top_n, holdings, capital)
10. PortfolioOptimizer.generate_trade_actions(optimization, holdings, cold_start_day)
11. RiskEngine.check_drawdown(portfolio_state)
    └─ If Tier 2: generate liquidation proposals → skip to step 13
12. RiskEngine.check_stop_losses(holdings, current_prices)
13. RiskEngine.evaluate_proposals(all_proposals, portfolio_state, market_data)
14. ExecutionEngine.submit_orders(approved_proposals, current_prices)
    └─ Trial mode: TelegramNotifier.request_approval() first
15. StateManager.save_portfolio_state(state)
16. StateManager.save_checkpoint_progress("cp1", "complete")
```

**Timeout**: 10 minutes (E-012). On timeout: abort, log, alert, allow next checkpoint.

### 2c. Hourly Monitoring — Mon-1 through Mon-5 (10:30 AM / 11:30 AM / 12:30 PM / 1:30 PM / 2:30 PM)

```
1. Build monitoring symbol scope = held symbols + today's baseline proposal symbols
2. DataPipeline.fetch_intraday_prices(monitor_symbols)
3. DataPipeline.fetch_sensex()
   └─ If >30% fail (E-008): skip, log, alert → EXIT
4. [Mon-1 only] ExecutionEngine.cancel_unfilled_orders()  # cleanup CP1 unfilled
5. RiskEngine.check_stop_losses(holdings, current_prices)
   └─ If triggered: ExecutionEngine.submit_orders(stop_loss_sells)
6. RiskEngine.check_drawdown(portfolio_state)
   └─ If Tier 2: ExecutionEngine.submit_orders(liquidation_sells)
7. RiskEngine.compute_volatility_regime(sensex_closes)
8. StateManager.save_portfolio_state(state)
9. StateManager.save_checkpoint_progress(checkpoint_id, "complete")
```

No new buy proposals. Only stop-losses, drawdown protection, and volatility regime monitoring. Same logic runs at each hourly monitoring checkpoint (Mon-1 through Mon-5), with unfilled order cleanup only at Mon-1. Max gap between any two checkpoints: 1 hour.

### 2d. Close (3:30 PM)

```
1. ExecutionEngine.cancel_hitl_proposals()
2. StateManager.save_checkpoint_progress("close", "complete")
```

**Error Handling**:
- Price fetch failure >30% (E-008): Skip entire checkpoint, log as unmonitored window
- Checkpoint timeout (E-012): Abort, alert, allow next checkpoint
- Telegram unreachable during HITL (E-009): Auto-execute stop-losses, auto-cancel others

**Component Dependencies**: DataPipeline, FeatureEngine, ModelManager, PortfolioOptimizer, RiskEngine, ExecutionEngine, StateManager, TelegramNotifier, ConfigManager

---

## 3. WeeklyService

**Purpose**: Orchestrate Saturday watchlist screening and Sunday model retraining.

### 3a. Saturday Screener (10:00 AM)

```
1. PortfolioOptimizer.get_top_n_history() → top_n_history
2. WatchlistManager.run_saturday_screener(top_n_history, candidate_pool)
3. TelegramNotifier.request_approval(screener_result.summary)
   └─ Sandbox/Trial: wait for approval (24-hour window, default: not applied)
   └─ Production: auto-apply with notification
4. WatchlistManager.apply_changes(approved_additions, approved_removals)
   └─ Additions trigger DataPipeline.bootstrap_history(new_symbols)
```

### 3b. Sunday Retrain (2:00 AM)

```
1. FeatureEngine.compute_features(qlib_data, full_watchlist, today)
2. ModelManager.train(feature_matrix, horizon)
   └─ On failure: ModelManager.load_model(fallback), TelegramNotifier.send_alert()
3. ModelManager.save_model(new_artifact_path)
```

**Component Dependencies**: PortfolioOptimizer, WatchlistManager, DataPipeline, FeatureEngine, ModelManager, TelegramNotifier, ConfigManager

---

## 4. StartupService

**Purpose**: System bootstrap, self-diagnostic, and crash recovery on process start.

**Trigger**: systemd process start (boot or restart).

**Orchestration Flow**:
```
1. ConfigManager.load_config()
2. ConfigManager.validate_config()
3. StateManager.run_self_diagnostic()
   └─ Check: DB integrity, schema version, bootstrap status,
      model artifact, config parseability, credentials, disk space, log writability
   └─ On critical failure: TelegramNotifier.send_alert(), EXIT
4. ModelManager.load_model()
   └─ If model missing/corrupt: TelegramNotifier.send_alert(WARN)
5. StateManager.load_portfolio_state()
6. If market hours:
   a. StateManager.get_completed_checkpoints(today) → completed
   b. Identify missed checkpoints
   c. RiskEngine.check_stop_losses(holdings, current_prices)  [safety check]
   d. Execute immediate stop-loss sells if triggered
   e. Resume from next scheduled checkpoint
7. HeartbeatService.send_trading_heartbeat() [confirm alive]
```

**Error Handling**: Critical diagnostic failure blocks entry to normal operation.

**Component Dependencies**: ConfigManager, StateManager, ModelManager, RiskEngine, ExecutionEngine, DataPipeline, TelegramNotifier

---

## 5. ShutdownService

**Purpose**: Graceful shutdown — complete in-flight operations and persist state.

**Trigger**: SIGTERM from systemd or manual shutdown command.

**Orchestration Flow**:
```
1. Signal all active services to stop accepting new work
2. Wait for in-flight orders to complete (30-second timeout — E-010)
3. ExecutionEngine.cancel_unfilled_orders() [any remaining]
4. StateManager.save_portfolio_state(current_state)
5. StateManager.save_checkpoint_progress("shutdown", timestamp)
6. Log shutdown completion to scheduler.log
```

**Component Dependencies**: ExecutionEngine, StateManager

---

## 6. HeartbeatService

**Purpose**: Send periodic alive signals so the trader knows the system is operational.

**Trigger**: Scheduler at 9:15 AM (trading days) and 12:00 PM (weekends/holidays).

**Orchestration Flow**:
```
Trading day:
  1. TelegramNotifier.send_heartbeat("AT alive — pre-market starting")

Weekend/Holiday:
  1. TelegramNotifier.send_heartbeat("AT alive — idle")
```

**Component Dependencies**: TelegramNotifier

---

## 7. BackupService

**Purpose**: Automated daily backup of critical state to Hetzner Storage Box.

**Trigger**: Scheduler at ~6:15 PM IST on trading days (after evening batch completes).

**Orchestration Flow**:
```
1. Export from StateManager: portfolio state, transaction log, audit trail,
   guardrail config, top-N history
2. Copy latest model artifact from ModelManager
3. Compress and transfer to Hetzner Storage Box (rsync over SSH or rclone)
4. Validate backup integrity
5. Prune backups older than 30 days
   └─ On failure: TelegramNotifier.send_alert()
```

**Component Dependencies**: StateManager, ModelManager, TelegramNotifier, ConfigManager

---

## Cross-Cutting Concerns

### Logging
All services log to **scheduler.log** (their own lifecycle events). Each component logs to its designated module log file (data.log, model.log, execution.log). See US-9.4.

### Error Propagation
Services catch component exceptions and decide whether to:
- **Retry** (transient failures like API timeouts)
- **Skip** (data unavailable — degrade gracefully)
- **Abort** (critical failures — halt and alert)
- **Fallback** (use previous state/model/plan)

### Timeout Enforcement
Each checkpoint service enforces a 10-minute timeout (E-012). On timeout, the service aborts cleanly and allows the next scheduled event to proceed.

### Idempotency
Services record completion in StateManager before exiting. On restart, the StartupService reads completion records to skip already-completed checkpoints and resume correctly.
