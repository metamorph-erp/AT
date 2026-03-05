# Deployment Architecture — Unit 1: Foundation & Data

> **Scope**: Deployment workflow, environment setup, and operational procedures  
> **Target**: Hetzner CPX32 VPS — single-machine deployment

---

## 1. Deployment Model

### 1.1 Strategy: Git Pull + systemd Restart (DC-4)

```
Developer Workstation (Windows)
    │
    │  git push
    ▼
GitHub Repository (private)
    │
    │  SSH → git pull
    ▼
Hetzner VPS (/opt/auto-trader/)
    │
    │  pip install -r requirements.txt (if deps changed)
    │  systemctl restart auto-trader
    ▼
Application Running (systemd managed)
```

**Deployment commands** (run via SSH):
```bash
cd /opt/auto-trader
git pull origin main
source venv/bin/activate
pip install -r requirements.txt --quiet
sudo systemctl restart auto-trader
sudo systemctl status auto-trader
```

### 1.2 Zero-Downtime Constraint

- Deploy only during **non-trading hours** (after 3:30 PM IST or weekends)
- No blue-green or rolling deployment — single process, single machine
- Restart takes < 5 seconds (Python import + config load + diagnostics)
- If bootstrap needed (rare): up to 15 minutes before trading flows resume

### 1.3 Rollback

```bash
cd /opt/auto-trader
git log --oneline -5          # Find previous commit
git checkout <prev-commit>    # Rollback (enters detached HEAD)
sudo systemctl restart auto-trader

# After verifying rollback works, if making it permanent:
# git checkout main && git reset --hard <prev-commit>
```

No database migration rollback needed — schema changes are manual and rare (Phase 1).

---

## 2. VPS Initial Setup

### 2.1 Setup Script: `deploy/setup.sh`

One-time VPS provisioning script:

```bash
#!/bin/bash
set -euo pipefail

# 1. Create dedicated user
sudo useradd -r -m -s /bin/bash autotrader

# 2. Install system dependencies
sudo apt update
sudo apt install -y python3.10 python3.10-venv python3-pip git logrotate ufw

# 3. Clone repository
sudo -u autotrader git clone git@github.com:<user>/auto-trader.git /opt/auto-trader

# 4. Create virtual environment
sudo -u autotrader python3.10 -m venv /opt/auto-trader/venv
sudo -u autotrader /opt/auto-trader/venv/bin/pip install -r /opt/auto-trader/requirements.txt

# 5. Create data and log directories
sudo -u autotrader mkdir -p /opt/auto-trader/data/qlib/bse_data
sudo -u autotrader mkdir -p /opt/auto-trader/data/models
sudo -u autotrader mkdir -p /opt/auto-trader/logs
sudo -u autotrader mkdir -p /opt/auto-trader/data/backup
chmod 700 /opt/auto-trader/data

# 6. Set timezone
sudo timedatectl set-timezone Asia/Kolkata

# 7. Configure swap (4 GB)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 8. Install systemd service
sudo cp /opt/auto-trader/deploy/systemd/auto-trader.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable auto-trader

# 9. Install logrotate config
sudo cp /opt/auto-trader/deploy/logrotate/auto-trader /etc/logrotate.d/

# 10. Install cron schedule
sudo cp /opt/auto-trader/deploy/cron/auto-trader-cron /etc/cron.d/

# 11. Configure UFW
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw --force enable

# 12. Configure unattended-upgrades
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# 13. SSH hardening
sudo sed -i '/^#\?PasswordAuthentication/c\PasswordAuthentication no' /etc/ssh/sshd_config
sudo sed -i '/^#\?PermitRootLogin/c\PermitRootLogin no' /etc/ssh/sshd_config
sudo systemctl restart sshd

echo "Setup complete. Create config/secrets.yaml manually, then: sudo systemctl start auto-trader"
```

### 2.2 Post-Setup Manual Steps

1. **Create `config/secrets.yaml`** with actual credentials (not in git)
2. **Set file permissions**: `chmod 600 /opt/auto-trader/config/secrets.yaml`
3. **Start service**: `sudo systemctl start auto-trader`
4. **Verify**: `sudo systemctl status auto-trader` + check `logs/scheduler.log`

---

## 3. Environment Configuration

### 3.1 Development Environment (Windows)

| Attribute | Value |
|---|---|
| OS | Windows 10/11 |
| Python | 3.10+ (installed via python.org) |
| Env | `python -m venv .venv` in project root |
| Database | SQLite in `data/autotrader.db` (same path, different machine) |
| Trading | Sandbox mode only — mock broker |
| Tests | `pytest tests/` — all tests run locally |
| Deployment | Not from Windows — push to git, SSH to VPS to pull |

### 3.2 Production Environment (Hetzner VPS)

| Attribute | Value |
|---|---|
| OS | Ubuntu 22.04 LTS |
| Python | 3.10 (system or deadsnakes PPA) |
| Env | `/opt/auto-trader/venv/` |
| Database | `/opt/auto-trader/data/autotrader.db` |
| Trading | Sandbox → Trial → Production (mode progression) |
| Process | systemd managed, auto-restart on failure |
| Monitoring | Telegram alerts + systemd journal |

### 3.3 Environment Differences

| Concern | Windows (Dev) | Linux (Prod) |
|---|---|---|
| Path separator | `\` (handled by `pathlib.Path`) | `/` |
| File permissions | No chmod equivalent (ignored) | 0o600/0o700 enforced |
| watchdog | Not used (no systemd) | sd_notify watchdog |
| cron | Not used (manual trigger) | System cron |
| logrotate | Not used (Python rotation only) | System + Python |
| Checkpoint timeout | `threading.Timer` (cooperative) | `signal.SIGALRM` (preemptive) |

---

## 4. Operational Procedures

### 4.1 Daily Operations (Automated)

| Time (IST) | Operation | Trigger |
|---|---|---|
| 09:10 | Pre-market health check | cron → Scheduler |
| 09:30 | CP1 checkpoint | cron → Scheduler → DataPipeline |
| 10:30–14:30 | Mon-1 to Mon-5 monitoring | cron → Scheduler → DataPipeline |
| 16:00 | Evening batch | cron → Scheduler → DataPipeline |
| 18:15 | Daily backup | cron → BackupService |

### 4.2 Manual Operations

| Operation | Command | When |
|---|---|---|
| View logs | `tail -f /opt/auto-trader/logs/data.log` | Debugging |
| Check status | `sudo systemctl status auto-trader` | Anytime |
| Force restart | `sudo systemctl restart auto-trader` | After crash |
| Deploy update | `git pull && sudo systemctl restart auto-trader` | Non-trading hours |
| Mode switch | Edit `config.yaml` → restart | Phase transition |
| DB inspection | `sqlite3 /opt/auto-trader/data/autotrader.db` | Debugging |

### 4.3 Incident Response

| Scenario | Detection | Response |
|---|---|---|
| Process crash | systemd auto-restart + Telegram alert | Check logs, fix root cause |
| VPS reboot | systemd auto-start | Verify via `systemctl status` |
| Disk > 80% | Telegram alert from daily check | Review logs/data, prune if needed |
| Database corruption | Startup diagnostic fails | Restore from latest backup |
| API key expired | HTTP 403 logged + retries exhausted | Update `secrets.yaml`, restart |

---

## 5. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Hetzner CPX32 VPS                         │
│                   Ubuntu 22.04 LTS                          │
│                   IST (Asia/Kolkata)                         │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              systemd: auto-trader.service              │  │
│  │                                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │         Python 3.10+ (venv)                     │  │  │
│  │  │                                                 │  │  │
│  │  │  ConfigManager ──► config/config.yaml           │  │  │
│  │  │       │              config/secrets.yaml         │  │  │
│  │  │       │              config/bse_holidays.yaml    │  │  │
│  │  │       ▼                                         │  │  │
│  │  │  StateManager ──► data/autotrader.db (SQLite)   │  │  │
│  │  │       │                                         │  │  │
│  │  │       ▼                                         │  │  │
│  │  │  DataPipeline ──► data/qlib/bse_data/           │  │  │
│  │  │       │                                         │  │  │
│  │  │  LoggerFactory ──► logs/*.log                   │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  cron ──► Scheduler trigger                                 │
│  logrotate ──► Log compression                              │
│  UFW ──► Firewall (deny incoming, allow outgoing)           │
│  unattended-upgrades ──► Security patches                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
         Outbound HTTPS (port 443)
                       │
          ┌────────────┼────────────────┐
          ▼            ▼                ▼
   AlphaVantage   BSE Website     Telegram API
   (market data)  (Bhavcopy CSV)  (alerts)

         Outbound SSH (port 23)
                       │
                       ▼
              Hetzner Storage Box
              (daily backups, €3/mo)
```
