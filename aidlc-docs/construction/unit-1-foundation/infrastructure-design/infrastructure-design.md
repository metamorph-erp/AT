# Infrastructure Design — Unit 1: Foundation & Data

> **Scope**: Infrastructure mapping for ConfigManager, StateManager, DataPipeline, Per-Module Logging, and supporting infrastructure components  
> **Target**: Hetzner CPX32 VPS (4 vCPU, 8 GB RAM, 160 GB NVMe, Ubuntu 22.04 LTS)  
> **Sources**: I-001 through I-013, NFR design patterns, logical components

---

## 1. Compute Infrastructure

### 1.1 VPS Specification

| Attribute | Value |
|---|---|
| Provider | Hetzner Cloud |
| Plan | CPX32 (shared AMD EPYC) |
| vCPU | 4 |
| RAM | 8 GB |
| Disk | 160 GB NVMe SSD |
| Swap | 4 GB (file-backed on NVMe) |
| OS | Ubuntu 22.04 LTS |
| Timezone | Asia/Kolkata (IST) |
| Cost | ~$12/month |

### 1.2 Python Runtime

| Attribute | Value |
|---|---|
| Python version | 3.10+ (Ubuntu 22.04 ships 3.10) |
| Virtual environment | `/opt/auto-trader/venv/` |
| Activation | Managed by systemd `ExecStart` — not manually activated |
| Package install | `pip install -r requirements.txt` inside venv |

### 1.3 Process Management — systemd

**Service file**: `deploy/systemd/auto-trader.service`

```ini
[Unit]
Description=Auto-Trader Quantitative Trading System
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=autotrader
Group=autotrader
WorkingDirectory=/opt/auto-trader
ExecStart=/opt/auto-trader/venv/bin/python -m src.main
Restart=on-failure
RestartSec=10
WatchdogSec=60
TimeoutStartSec=1200
Environment=PYTHONUNBUFFERED=1

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/opt/auto-trader/data /opt/auto-trader/logs
ProtectHome=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

**Key properties**:
- `Restart=on-failure` + `RestartSec=10` → auto-restart after crash (I-010)
- `WatchdogSec=60` → systemd kills process if no watchdog ping in 60s (I-010)
- `TimeoutStartSec=1200` → allows up to 20 min for first-run bootstrap and startup diagnostics
- `User=autotrader` → dedicated non-root user
- `ReadWritePaths` → only `data/` and `logs/` are writable (least privilege)

### 1.4 Scheduling — cron

**Cron file**: `deploy/cron/auto-trader-cron` (installed to `/etc/cron.d/`)

Unit 1 does not own the scheduler, but its DataPipeline is invoked by cron-triggered scheduler (Unit 4). The cron schedule is:

| Time (IST) | Job | Unit 1 Involvement |
|---|---|---|
| 16:00 Mon-Fri | Evening batch | DataPipeline: Bhavcopy fetch, Sensex fetch, Qlib append |
| 09:30 Mon-Fri | CP1 | DataPipeline: Intraday price fetch (held + proposed) |
| 10:30-14:30 hourly | Mon-1 to Mon-5 | DataPipeline: Intraday price refresh |
| Sat 10:00 | Weekly screener | DataPipeline: Liquidity data for watchlist refresh |
| Sun 02:00 | Weekly retrain | DataPipeline: Provides training data (Qlib binary store) |

---

## 2. Storage Infrastructure

### 2.1 Filesystem Layout

```
/opt/auto-trader/                    # Application root
├── config/
│   ├── config.yaml                  # Main config (0o644)
│   ├── secrets.yaml                 # Credentials (0o600, in .gitignore)
│   ├── bse_holidays.yaml            # BSE calendar (0o644)
│   └── logging.yaml                 # Optional logging overrides
├── src/                             # Application code (read-only at runtime)
│   ├── components/
│   ├── services/
│   ├── infrastructure/              # RateLimiter, RetryHandler, etc.
│   └── models/types.py
├── data/                            # Persistent data (0o700, autotrader user)
│   ├── autotrader.db                # SQLite database (0o600)
│   ├── autotrader.db-wal            # WAL file (auto-managed by SQLite)
│   ├── autotrader.db-shm            # Shared memory file (auto-managed)
│   ├── qlib/                        # Qlib binary data store
│   │   └── bse_data/                # Per-instrument binary files
│   └── models/                      # ML model artifacts (Unit 2)
│       └── lgbm_model_YYYYMMDD.bin
├── logs/                            # Log files (0o755, autotrader user)
│   ├── data.log                     # DataPipeline + adapters
│   ├── model.log                    # FeatureEngine + PredictionModel (Unit 2)
│   ├── execution.log                # ExecutionEngine + RiskManager (Unit 3)
│   └── scheduler.log                # Scheduler + orchestration (Unit 4)
├── tests/                           # Test code (not deployed)
├── deploy/                          # Deployment configs
│   ├── systemd/auto-trader.service
│   ├── logrotate/auto-trader
│   ├── cron/auto-trader-cron
│   └── setup.sh
├── requirements.txt
├── requirements-dev.txt
└── .gitignore
```

### 2.2 SQLite Database

| Property | Value |
|---|---|
| Path | `data/autotrader.db` |
| Mode | WAL (set at connection open) |
| File permissions | 0o600 (owner read/write only) |
| Max expected size | < 100 MB (years of operation) |
| Backup method | SQLite backup API → compressed copy to Storage Box |

### 2.3 Qlib Data Store

| Property | Value |
|---|---|
| Path | `data/qlib/bse_data/` |
| Format | Qlib binary (`dump_bin`) |
| Growth | ~50 rows/day (50 symbols × 1 OHLCV row) |
| Max size | < 500 MB for 5 years of 50 stocks |
| Initialization | `qlib.init(provider_uri="data/qlib/bse_data")` |

### 2.4 Disk Budget

| Category | Size | Notes |
|---|---|---|
| OS + packages | ~5 GB | Ubuntu 22.04 + Python + pip packages |
| Swap | 4 GB | File on NVMe |
| SQLite DB | < 100 MB | Grows trivially |
| Qlib data | < 500 MB | 5-year projection |
| Logs (uncompressed) | < 500 MB | 30-day retention, post-2-day compression |
| Model artifacts | < 200 MB | Last 4 weekly models |
| **Total used** | **~10 GB** | **Leaves 150 GB free on 160 GB NVMe** |

---

## 3. Logging Infrastructure

### 3.1 Application-Level (Python)

Handled by `LoggerFactory` (from NFR Design):
- 4 module-specific file handlers write to `logs/{module}.log`
- No in-process rotation in the application layer
- `CredentialScrubber` filter on all handlers

### 3.2 System-Level — logrotate

**Config file**: `deploy/logrotate/auto-trader` (installed to `/etc/logrotate.d/`)

```
/opt/auto-trader/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    copytruncate
    dateext
    dateformat -%Y%m%d
}
```

**Key settings**:
- `delaycompress` → compress after 2 days (keeps yesterday readable without zcat)
- `copytruncate` → rotates active files without requiring application restart
- `rotate 30` → 30-day retention
- Runs via system cron (`/etc/cron.daily/logrotate`)

**Rotation ownership**: `logrotate` is the single owner for rotate/compress/retention to match I-012 exactly. The Python layer only writes logs and does not rotate them.

### 3.3 Disk Space Monitoring

- Startup diagnostic (E-011 check 7): `shutil.disk_usage("/")` — warn if > 80% used
- Telegram alert if NVMe exceeds 80% usage threshold (O-002)
- No automated cleanup — 150 GB headroom makes this a notification-only concern

---

## 4. Networking Infrastructure

### 4.1 UFW Firewall Rules (I-011)

```bash
# Default deny incoming
ufw default deny incoming
ufw default allow outgoing

# SSH (management)
ufw allow 22/tcp

# No inbound ports needed — all outbound:
# - Indian Stock Market API: HTTPS 443 outbound
# - BSE Bhavcopy: HTTPS 443 outbound
# - Telegram API: HTTPS 443 outbound
# - Hetzner Storage Box: SSH 23 outbound (for rsync backup)

ufw enable
```

**OpenAlgo**: localhost-only (127.0.0.1:5000) — no firewall rule needed, not exposed externally.

### 4.2 Outbound API Connections (Unit 1)

| Service | URL Pattern | Protocol | Port | Rate Limit |
|---|---|---|---|---|
| Indian Stock Market API | `https://raw.githubusercontent.com/...` | HTTPS | 443 | Self-imposed: 2/s (0.5s spacing) |
| BSE Bhavcopy | `https://www.bseindia.com/download/BhscsrUpd/...` | HTTPS | 443 | 1 call/evening |
| Telegram Bot | `https://api.telegram.org/bot{token}/...` | HTTPS | 443 | Burst OK |

### 4.3 SSH Access (I-011)

| Property | Value |
|---|---|
| Authentication | Key-only (password disabled) |
| Root login | Disabled (`PermitRootLogin no`) |
| Port | 22 (standard) |
| Purpose | Deployment (`git pull`), maintenance, debugging |

---

## 5. Security Infrastructure

### 5.1 OS-Level Hardening (I-011)

| Measure | Implementation |
|---|---|
| SSH key-only auth | `/etc/ssh/sshd_config`: `PasswordAuthentication no` |
| Dedicated user | `autotrader` (non-root, no sudo) |
| Auto security patches | `unattended-upgrades` — security-only, auto-reboot Sun 3:00 AM (I-011a) |
| Firewall | UFW with deny-incoming default |

### 5.2 File Permissions

| Path | Owner | Permissions | Rationale |
|---|---|---|---|
| `/opt/auto-trader/` | autotrader:autotrader | 0o755 | Application root |
| `config/` | autotrader:autotrader | 0o750 | Config directory — contains secrets.yaml |
| `config/secrets.yaml` | autotrader:autotrader | 0o600 | Credential file — owner only |
| `data/autotrader.db` | autotrader:autotrader | 0o600 | Database — owner only |
| `data/` | autotrader:autotrader | 0o700 | Data directory — owner only |
| `logs/` | autotrader:autotrader | 0o755 | Logs readable for debugging |
| `src/` | autotrader:autotrader | 0o755 | Code — read-only at runtime |

### 5.3 Credential Protection (Phase 1)

- `config/secrets.yaml` — plaintext, `.gitignore`'d, file perm 0o600
- Phase 2: Fernet symmetric encryption with machine-derived key
- In-memory cache after first read — never re-read from disk
- Log scrubbing via `CredentialScrubber` on all loggers

---

## 6. Backup Infrastructure (I-013)

### 6.1 Backup Target

| Property | Value |
|---|---|
| Service | Hetzner Storage Box |
| Cost | ~€3/month |
| Protocol | rsync over SSH (port 23) |
| Retention | 30 days |
| Schedule | Daily ~6:15 PM IST (after evening batch) |

### 6.2 Backup Scope

| Item | Method | Size |
|---|---|---|
| SQLite DB | `sqlite3.backup()` API → temp copy → compress → rsync | < 10 MB |
| Model artifact | Direct file copy → compress → rsync | < 50 MB |
| Config files | Direct copy → rsync | < 1 KB |
| **Total daily upload** | | **< 60 MB compressed** |

### 6.3 Backup Flow

```
backup_flow():
    1. WAL checkpoint: PRAGMA wal_checkpoint(TRUNCATE) → clean DB state
    2. SQLite backup: sqlite3.backup() → data/backup/autotrader_YYYYMMDD.db
    3. Compress: gzip backup + current model artifact
    4. rsync to Storage Box: rsync -az --delete-after -e "ssh -p 23" data/backup/ user@storageboxXXX.your-storagebox.de:auto-trader/
    5. Prune local temp: delete data/backup/ after successful transfer
    6. Log result + Telegram notification (success or failure)
```

**Note**: BackupService is in Unit 4. Unit 1's StateManager provides `PRAGMA wal_checkpoint` and the `sqlite3.backup()` API support method.

---

## 7. Monitoring Infrastructure

### 7.1 systemd Watchdog (I-010)

- `WatchdogSec=60` in service file
- Application must call `sd_notify("WATCHDOG=1")` every < 60s
- On timeout: systemd kills and restarts the process
- Python implementation: `sdnotify` package or direct socket write via `os.environ.get("NOTIFY_SOCKET")`

### 7.2 System Monitoring

| What | How | Alert |
|---|---|---|
| Process alive | systemd auto-restart | Auto-handled |
| Disk usage > 80% | `shutil.disk_usage()` at startup + daily check | Telegram |
| Checkpoint failures | Checkpoint log → status = FAILED | Telegram |
| API rate limit errors | Logged in `data.log` at WARNING | No alert (self-healing via retry) |
| Database corruption | `PRAGMA integrity_check` at startup | Telegram + exit |

### 7.3 No External Monitoring Stack

- No Prometheus, Grafana, or ELK stack — overkill for single-process system
- Application-level logging + Telegram alerts provide adequate observability
- `tail -f logs/*.log` for real-time debugging via SSH
