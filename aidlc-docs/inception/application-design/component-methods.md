# Component Methods — Auto-Trader (AT)

> **Scope**: Phase 1 method signatures. Phase 2 additions noted where relevant.  
> **Note**: Detailed business rules, validation logic, and algorithm internals are deferred to **Functional Design** (per-unit, CONSTRUCTION phase). This document defines *what* each component exposes, not *how* it computes results.

---

## 1. DataPipeline

```
fetch_eod_data(symbols: list[str], date: date) -> dict[str, OHLCVRecord]
    Fetch end-of-day OHLCV for given symbols from BSE Bhavcopy.
    Returns validated records keyed by symbol. Raises on source unavailable.

fetch_intraday_prices(symbols: list[str]) -> dict[str, IntradayBar]
    Fetch current intraday partial bars for given symbols.
    Phase 1 source: AlphaVantage. Phase 2 source: Kotak Neo API.
    Returns validated bars. Applies D-005a rejection rules.

fetch_sensex() -> float
    Fetch current Sensex index level.

fetch_sensex_history(days: int = 20) -> list[float]
    Return last N days of Sensex closing prices (for volatility regime computation).

bootstrap_history(symbols: list[str], years: int = 2) -> BootstrapResult
    Bulk-download historical OHLCV for all symbols from AlphaVantage.
    Returns download status per symbol. Blocks until complete.

load_into_qlib(raw_data: dict[str, OHLCVRecord]) -> None
    Load validated raw data into Qlib dataset structure.
    Handles date alignment, holiday gaps, rolling window maintenance.

get_trading_calendar(year: int) -> list[date]
    Return list of BSE trading days (excludes weekends and holidays).

is_trading_day(date: date) -> bool
    Check if given date is a BSE trading day.

validate_data(records: dict[str, OHLCVRecord]) -> ValidationResult
    Apply D-005/D-005a validation rules. Returns accepted/rejected split.
```

**Key Types**:
- `OHLCVRecord`: open, high, low, close, volume, date, symbol
- `IntradayBar`: open, high, low, close, volume, timestamp, symbol
- `BootstrapResult`: per-symbol status (success/failed/gap_count)
- `ValidationResult`: accepted records, rejected records with rejection reasons

---

## 2. FeatureEngine

```
compute_features(qlib_data: QlibDataset, symbols: list[str], date: date) -> FeatureMatrix
    Compute full technical feature set for given symbols up to given date.
    Returns feature matrix (stocks × features × dates).

compute_incremental_features(qlib_data: QlibDataset, symbols: list[str], 
                              intraday_bars: dict[str, IntradayBar]) -> FeatureMatrix
    Compute features using partial intraday bar (for CP1 inference).
    Updates last row of feature matrix without full recomputation.

get_feature_names() -> list[str]
    Return ordered list of feature names in the feature matrix.

validate_feature_stability(full_features: FeatureMatrix, 
                            partial_features: FeatureMatrix) -> StabilityReport
    Compare full-bar vs partial-bar feature values for stability assessment (DC-7).
```

**Key Types**:
- `FeatureMatrix`: numpy array (stocks × features × dates) with metadata
- `StabilityReport`: per-feature deviation metrics, overall stability flag

---

## 3. ModelManager

```
train(feature_matrix: FeatureMatrix, horizon: int = 5) -> TrainResult
    Full LightGBM retrain on rolling 2-year window.
    Persists model artifact to disk. Logs metrics. Returns training summary.

predict(feature_matrix: FeatureMatrix) -> dict[str, float]
    Run inference using current model artifact.
    Returns alpha score (expected N-day return) per symbol.
    Raises on anomalous output detection (M-007).

load_model(artifact_path: str | None = None) -> None
    Load model artifact from disk. Default: latest. Fallback: last known good.

save_model(artifact_path: str) -> None
    Persist current model artifact to given path.

get_model_info() -> ModelInfo
    Return current model metadata: version, train date, metrics, staleness days.

is_stale(warn_days: int = 14, halt_days: int = 21) -> StalenessStatus
    Check model staleness. Returns status: OK, WARN (>14d), or HALT (>21d).
```

**Key Types**:
- `TrainResult`: model_version, accuracy, sharpe, train_duration, artifact_path
- `ModelInfo`: version, train_date, metrics_dict, artifact_path, staleness_days
- `StalenessStatus`: enum (OK, WARN, HALT)

---

## 4. PortfolioOptimizer

```
rank_stocks(alpha_scores: dict[str, float]) -> list[RankedStock]
    Rank all stocks by alpha score descending.
    Returns ordered list with rank and score.

optimize_portfolio(ranked_stocks: list[RankedStock], top_n: int,
                    current_holdings: dict[str, Holding],
                    total_capital: float) -> OptimizationResult
    Run Qlib MVO on top-N stocks.
    Returns target weights, share quantities, and trade actions.
    Considers volatility, cross-stock correlations, portfolio risk.

generate_trade_actions(optimization: OptimizationResult,
                        current_holdings: dict[str, Holding],
                        cold_start_day: int = 0) -> list[TradeProposal]
    Convert optimisation output to concrete buy/sell proposals.
    Applies S-009 cold-start ramp. Skips re-buys for retained stocks (S-008).
    Applies M-008 3-day hold for ranking-based exits.

get_top_n_history() -> list[TopNSnapshot]
    Return historical top-N membership snapshots (RP-008).
```

**Key Types**:
- `RankedStock`: symbol, alpha_score, rank
- `Holding`: symbol, quantity, avg_entry_price, entry_date, highest_seen_price
- `OptimizationResult`: target_weights dict, share_quantities dict, expected_risk
- `TradeProposal`: symbol, action (BUY/SELL), quantity, reason, source (ranking/stop-loss/drawdown)
- `TopNSnapshot`: date, symbols list

---

## 5. RiskEngine

```
evaluate_proposals(proposals: list[TradeProposal],
                    portfolio_state: PortfolioState,
                    market_data: MarketSnapshot) -> list[EvaluatedProposal]
    Run all guardrails (G-001 to G-013) against trade proposals.
    Returns proposals annotated with pass/fail per guardrail.

check_position_sizing(proposal: TradeProposal, 
                       portfolio_state: PortfolioState) -> GuardrailResult
    Evaluate G-001, G-002, G-003 against a single proposal.

check_drawdown(portfolio_state: PortfolioState) -> DrawdownStatus
    Evaluate G-004 (daily 3%), G-005 Tier 1 (10%), G-005 Tier 2 (15%).
    Returns current drawdown level and active restrictions.

check_stop_losses(holdings: dict[str, Holding],
                   current_prices: dict[str, float]) -> list[TradeProposal]
    Check trailing stop-loss (G-006) for all holdings.
    Returns sell proposals for triggered stops.

compute_volatility_regime(sensex_closes: list[float]) -> VolatilityRegimeStatus
    Compute 20-day annualized realized vol from Sensex daily returns.
    Evaluate G-007 (>30% halve) and G-008 (>45% halt buys).

check_circuit_breaker(sensex_change_pct: float) -> bool
    Evaluate G-009: Sensex ≥10% intraday drop.

get_daily_proposal_count() -> int
    Return proposals submitted today (for G-010 enforcement).

apply_volatility_adjustment(proposals: list[TradeProposal],
                      regime: VolatilityRegimeStatus) -> list[TradeProposal]
    Halve position sizes if Sensex realized vol > 30%.
```

**Key Types**:
- `EvaluatedProposal`: proposal, approved (bool), guardrail_results list
- `GuardrailResult`: guardrail_id, passed (bool), reason, values
- `DrawdownStatus`: daily_pct, total_pct, tier (NONE/DAILY_HALT/TIER1/TIER2), cooling_off_remaining
- `VolatilityRegimeStatus`: enum (NORMAL, HALVE, HALT_BUYS)
- `PortfolioState`: holdings, cash, total_capital, peak_capital, daily_start_value
- `MarketSnapshot`: prices, sensex, sensex_closes_20d, sensex_change_pct

---

## 6. ExecutionEngine

```
submit_orders(proposals: list[EvaluatedProposal],
               current_prices: dict[str, float]) -> list[OrderResult]
    Submit approved proposals to OpenAlgo as limit orders (buys at LTP+0.3%, sells at LTP-0.3%).
    Stop-loss exits submitted as market orders.
    Returns order submission results.

cancel_unfilled_orders() -> list[OrderResult]
    Cancel all unfilled limit orders from previous checkpoint (IM-006a).

cancel_hitl_proposals() -> list[OrderResult]
    Cancel any proposals still awaiting HITL approval at 3:30 PM (IM-010).

get_order_status(order_ids: list[str]) -> dict[str, OrderStatus]
    Check current status of submitted orders.

retry_queued_orders() -> list[OrderResult]
    Process persistent on-disk order queue (E-002 backoff).
    Re-validates limit price on each retry attempt.

validate_pre_order(proposal: EvaluatedProposal,
                    current_prices: dict[str, float]) -> PreOrderCheck
    Validate: stock held for sells (X-006), liquidity for buys (X-009).
```

**Key Types**:
- `OrderResult`: order_id, symbol, action, quantity, price, status, timestamp
- `OrderStatus`: enum (SUBMITTED, FILLED, PARTIAL, REJECTED, CANCELLED)
- `PreOrderCheck`: valid (bool), rejection_reason

---

## 7. WatchlistManager

```
seed_watchlist(liquidity_threshold: float = 5_000_000) -> list[str]
    Initialise from BSE 100 filtered by X-009 liquidity threshold.
    Trigger bootstrap_history for all seed stocks.
    Returns list of added symbols.

run_saturday_screener(top_n_history: list[TopNSnapshot],
                       candidate_pool: list[str]) -> ScreenerResult
    Evaluate additions (volume criterion) and removals (absent ≥4 weeks from top-N).
    Returns proposed changes for approval.

apply_changes(approved_additions: list[str], 
               approved_removals: list[str]) -> None
    Apply approved watchlist changes. Trigger data download for additions (W-009).

check_suspended_stocks(symbols: list[str]) -> list[SuspensionEvent]
    Detect suspended/delisted/circuit-halted stocks > 3 trading days (W-007).

freeze_holding(symbol: str) -> None
    Mark position as frozen: excluded from re-ranking, stop-loss continues.

unfreeze_and_liquidate(symbol: str) -> TradeProposal
    On stock resume, generate market sell to liquidate frozen position.

get_active_watchlist() -> list[str]
    Return current active watchlist symbols.

get_frozen_holdings() -> list[str]
    Return symbols with frozen holdings.
```

**Key Types**:
- `ScreenerResult`: additions (list[str]), removals (list[str]), reasoning per symbol
- `SuspensionEvent`: symbol, event_type, days_suspended, detected_date

---

## 8. TelegramNotifier

```
send_message(text: str, parse_mode: str = "HTML") -> bool
    Send a message to the configured chat. Returns success status.
    Queues locally if unreachable, retries every 5 min.

send_report(report: FormattedReport) -> bool
    Send a formatted report (daily summary, backtest results, etc.).

send_heartbeat(message: str) -> bool
    Send a brief heartbeat message.

send_alert(alert_type: str, details: str) -> bool
    Send a guardrail trigger, error, or system alert.

request_approval(request_id: str, message: str, 
                  timeout_minutes: int = 15) -> ApprovalResponse
    Send approval request and wait for response within timeout.
    On timeout: auto-execute stop-losses, auto-cancel other proposals (E-009).

check_pending_approvals() -> list[ApprovalResponse]
    Check status of any pending HITL approval requests.
```

**Key Types**:
- `FormattedReport`: title, sections list, raw_text
- `ApprovalResponse`: request_id, approved (bool | None), response_text, timed_out (bool)

---

## 9. ReportGenerator

```
generate_daily_report(portfolio_state: PortfolioState,
                       trades: list[OrderResult],
                       guardrail_log: list[GuardrailResult],
                       model_info: ModelInfo) -> FormattedReport
    Produce daily Telegram report: snapshot, P&L, trades, guardrails, model accuracy.

generate_audit_entry(trade: OrderResult, model_info: ModelInfo,
                      features: FeatureMatrix, guardrails: list[GuardrailResult],
                      portfolio_state: PortfolioState) -> AuditEntry
    Produce full audit trail entry per RP-005a.

generate_backtest_report(results: BacktestResults) -> FormattedReport
    Format backtest outcome: return, Sharpe, DD, win rate, costs, vs Sensex.

generate_config_change_log(changes: list[ConfigChange]) -> FormattedReport
    Format guardrail config change summary per G-012.
```

**Key Types**:
- `AuditEntry`: timestamp, model_version, feature_snapshot_ref, guardrail_results, portfolio_snapshot, trade_details
- `ConfigChange`: timestamp, key, old_value, new_value, mode

---

## 10. StateManager

```
save_portfolio_state(state: PortfolioState) -> None
    Persist current portfolio state (holdings, cash, P&L) to SQLite.

load_portfolio_state() -> PortfolioState
    Load last persisted portfolio state. Returns empty state if none exists.

save_checkpoint_progress(checkpoint_id: str, status: str) -> None
    Record checkpoint completion status (for crash recovery).

get_completed_checkpoints(date: date) -> list[str]
    Return checkpoint IDs completed today (for market-hours restart — E-007).

save_baseline_plan(plan: BaselinePlan) -> None
    Persist evening batch baseline plan.

load_baseline_plan() -> BaselinePlan | None
    Load latest baseline plan.

save_order(order: OrderResult) -> None
    Persist order to transaction log.

save_audit_entry(entry: AuditEntry) -> None
    Persist audit trail entry.

save_drawdown_state(state: DrawdownState) -> None
    Persist drawdown tracking state (daily P&L, peak, tier).

load_drawdown_state() -> DrawdownState
    Load drawdown state for current period.

save_config_change(change: ConfigChange) -> None
    Persist configuration change event.

save_top_n_snapshot(snapshot: TopNSnapshot) -> None
    Persist top-N membership for RP-008 history.

run_self_diagnostic() -> DiagnosticResult
    E-011: Check model artifact, portfolio state, OpenAlgo reachability,
    Telegram reachability, disk space, NTP clock sync.
```

**Key Types**:
- `BaselinePlan`: date, ranked_stocks, trade_proposals, vol_regime, plan_status
- `DrawdownState`: peak_capital, daily_start_value, tier, cooling_off_start, cooling_off_remaining
- `DiagnosticResult`: checks dict (name → pass/fail/warning), overall_status

---

## 11. ConfigManager

```
load_config(config_path: str = "config.yaml") -> Config
    Load and validate configuration from disk.

get_mode() -> OperatingMode
    Return current operating mode (Sandbox/Trial/Production).

get_guardrail(guardrail_id: str) -> GuardrailConfig
    Return guardrail threshold for current mode.

update_guardrail(guardrail_id: str, new_value: Any, reason: str) -> ConfigChange
    Update guardrail value with change logging (G-012).

get_api_credentials(service: str) -> dict[str, str]
    Return decrypted API credentials for given service.

get_log_level(module: str) -> str
    Return configured log level for given module (DEBUG/INFO/WARNING/ERROR/CRITICAL).

get_model_params() -> dict[str, Any]
    Return LightGBM hyperparameters and prediction horizon.

get_top_n_count() -> int
    Return configured top-N selection count.

validate_config() -> list[str]
    Validate all configuration values. Returns list of warnings/errors.
```

**Key Types**:
- `Config`: full parsed configuration object
- `OperatingMode`: enum (SANDBOX, TRIAL, PRODUCTION)
- `GuardrailConfig`: guardrail_id, value, mode, description

---

## 12. BacktestEngine

```
run_backtest(start_date: date, end_date: date,
              initial_capital: float = 50_000) -> BacktestResults
    Run full pipeline simulation over historical data.
    Exercises: features → model → ranking → MVO → guardrails → simulated fills.
    Applies transaction cost model and slippage.

evaluate_viability(results: BacktestResults) -> ViabilityAssessment
    Check backtest results against AI-recommended viability thresholds (Q9).
    Returns pass/fail with detailed breakdown.
```

**Key Types**:
- `BacktestResults`: annualised_return, sharpe_ratio, max_drawdown, win_rate, avg_holding_days, total_costs, vs_sensex_return, equity_curve, trades_log
- `ViabilityAssessment`: passed (bool), metrics vs thresholds table, recommendation
