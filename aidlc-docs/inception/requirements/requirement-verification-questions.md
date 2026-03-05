# Requirements Verification Questions

## Context
The formal requirements document (Requirements_Formal.md v3.7) is exceptionally comprehensive with 638 lines covering 15 sections. The following questions address genuine gaps and ambiguities that need resolution before system design and implementation.

Please answer each question by placing the letter (A, B, C, etc.) after the `[Answer]:` tag.

---

## Question 1
**Indian Stock Market API — GitHub Repository Identification**

Assumption 9 and multiple requirements (D-002, D-006, IM-003) reference "Indian Stock Market API (GitHub)" as the sole intraday price source. Which specific GitHub repository should be used?

A) `datawizz/indian-stock-market-api` — REST API wrapper with real-time BSE/NSE data
B) Another specific repository you already have in mind (describe below)
C) I need you to research and recommend the best open-source option fitting the requirements (real-time BSE/NSE, no API key, free)
X) Other (please describe after [Answer]: tag below)

[Answer]: X: Lets use these APIs: https://www.alphavantage.co for research and Kotak neo api for realtime prices and trading.

---

## Question 2
**Seed Watchlist — Initial Stock Selection**

W-002 states the system starts with a "manually configured seed list." What stocks should be in the initial seed watchlist for Sandbox mode?

A) I will provide a specific list of BSE stock symbols (describe below)
B) Use the BSE Sensex 30 components as the initial seed (well-known, liquid stocks)
C) Use the BSE 100 index filtered down to ~30-40 stocks meeting the X-009 liquidity threshold (BSE average daily value > ₹50 lakh)
D) Start with a small test set of 10-15 well-known large-cap BSE stocks for initial development, expand later
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 3
**State Persistence Format (DC-3)**

DC-3 identifies state persistence format as an open design decision. Portfolio state, baseline plans, audit trails, and checkpoint logs all need persistence. What is your preference?

A) **SQLite** — single-file database, supports queries and atomic transactions, ideal for structured state + audit trails; slightly less human-readable
B) **JSON files** — human-readable, easy debugging, simple to implement; less robust for concurrent access and queries
C) **Hybrid** — SQLite for portfolio state, transaction logs, and audit trails (query-friendly); JSON for configuration, baseline plans, and human-readable artifacts
D) Let the AI decide based on best engineering practice for this system
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 4
**Deployment Strategy (DC-4)**

DC-4 identifies deployment as an open design decision. How should new code reach the Hetzner VPS?

A) **Git pull + systemd restart** — VPS has a git clone; deploy by SSH → git pull → systemd restart; simple, auditable
B) **Docker container** — containerized deployment with docker-compose; reproducible, isolated, easy rollback
C) **rsync + systemd restart** — direct file sync from development machine; simple but less auditable
D) Let the AI decide based on best engineering practice for a single-VPS system
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 5
**Candidate Pool — Initial Composition**

W-003 references a "user-maintained candidate pool" of 50-150 pre-approved stocks for the Saturday screener. How should this candidate pool be initially populated?

A) I will provide a specific CSV/list of candidate stock symbols
B) Use the BSE 200 or BSE 500 index constituents filtered by the X-009 liquidity threshold
C) Start with BSE Sensex 30 + BSE MidCap 50 as the initial pool (~80 stocks)
D) Defer the candidate pool — start with just the seed watchlist and add the candidate pool screener later
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 6
**Broker Selection for Phase 2**

Assumption 14 states a broker will be selected before Phase 2. Do you have a broker preference, or should the system be designed broker-agnostic with OpenAlgo?

A) **Zerodha** — largest Indian broker, well-documented API (Kite Connect), ₹20/order flat
B) **Angel One** — free API (SmartAPI), no per-order charges for delivery, good for small capital
C) **Upstox** — free API, competitive pricing
D) No preference yet — design for OpenAlgo broker-agnosticism first; broker will be selected later
X) Other (please describe after [Answer]: tag below)

[Answer]: X Kotak neo

---

## Question 7
**Telegram Bot — Existing or New?**

The system requires a Telegram Bot for notifications (all modes) and HITL approvals (Trial mode). Do you already have a Telegram bot set up?

A) Yes — I have an existing Telegram bot token and chat ID ready
B) No — create a new Telegram bot as part of Phase 1 setup; provide instructions
C) I'll set up the Telegram bot myself; just tell me what permissions/commands it needs
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 8
**Python Project Structure**

The technology stack prescribes Python 3.10+. What project structure approach do you prefer?

A) **Flat module structure** — single Python package with submodules (e.g., `at/data/`, `at/model/`, `at/risk/`, `at/execution/`)
B) **Poetry-managed project** — modern Python packaging with `pyproject.toml`, dependency management, and virtual environments
C) **Simple pip + requirements.txt** — minimal tooling, `venv` for isolation
D) Let the AI decide based on best practice for a production Python system on a VPS
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 9
**Backtesting Minimum Viability Gate (P1-017)**

P1-017 specifies minimum viability criteria: Sharpe ≥ 0.3, max drawdown < 25%. These are for the historical backtest only (not live Sandbox). Are you comfortable with these thresholds, or would you like to adjust them?

A) Keep as-is — Sharpe ≥ 0.3, max drawdown < 25% (conservative entry gate)
B) Raise the bar — Sharpe ≥ 0.5, max drawdown < 20% (stricter validation)
C) Lower the bar — Sharpe ≥ 0.2, max drawdown < 30% (more lenient, allows earlier Sandbox entry)
D) Let the AI recommend appropriate thresholds based on BSE market characteristics
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 10
**LightGBM Prediction Horizon and Feature Set Review**

M-002 specifies a default 5-day prediction horizon. F-001 lists initial features (momentum, volatility, RSI, MACD, volume spikes, MA, relative strength vs Sensex). Do you want to:

A) Keep the defaults — 5-day horizon with the listed feature set; tune during backtesting
B) Start with a broader feature set (add Bollinger Bands, ATR, OBV, ADX, etc.) and use feature importance to prune
C) Start minimal (3-4 features: momentum + RSI + volume) and incrementally add
D) Let the AI design an optimal feature set based on LightGBM + equity swing trading best practices
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 11
**OpenAlgo Version and Self-Hosting**

X-001/X-003 require self-hosting OpenAlgo. Have you verified that OpenAlgo supports mock/paper order mode out of the box for Phase 1?

A) Yes — confirmed that OpenAlgo supports paper trading mode
B) No — need to verify this during development; if not supported, implement a mock execution layer internally
C) Not sure — please research and confirm during the design phase
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 12
**Development Environment**

I-009 specifies local development before Hetzner deployment. What is your local development environment?

A) Windows with WSL2 (Linux subsystem)
B) Windows native (PowerShell / CMD)
C) macOS
D) Linux (native)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---
