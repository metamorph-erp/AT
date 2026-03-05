# Infrastructure Design Plan — Unit 1: Foundation & Data

## Plan Overview
Map Unit 1 logical components to actual infrastructure services on Hetzner CPX32 VPS.

## Execution Steps

- [x] **Step 1**: Map compute infrastructure — Python runtime, systemd service, process management
- [x] **Step 2**: Map storage infrastructure — SQLite file location, Qlib data directory, config files
- [x] **Step 3**: Map logging infrastructure — logrotate configuration, log directory, rotation schedule
- [x] **Step 4**: Map networking infrastructure — AlphaVantage HTTPS outbound, BSE Bhavcopy download, UFW rules
- [x] **Step 5**: Map deployment infrastructure — git-based deployment, VPS setup script, directory layout
- [x] **Step 6**: Map backup infrastructure — Hetzner Storage Box, rsync, SQLite backup API

## Artifacts to Generate
1. `aidlc-docs/construction/unit-1-foundation/infrastructure-design/infrastructure-design.md`
2. `aidlc-docs/construction/unit-1-foundation/infrastructure-design/deployment-architecture.md`

---

## Questions

**Assessment**: All infrastructure decisions are fully specified in prior artifacts:

- **Compute**: Hetzner CPX32 (4 vCPU, 8 GB RAM, 160 GB NVMe), Ubuntu 22.04 LTS, Python 3.10+ venv, systemd with Restart=on-failure + watchdog (I-010)
- **Storage**: SQLite at `data/autotrader.db`, Qlib at `data/qlib/`, config at `config/`, model artifacts at `data/models/`
- **Logging**: logrotate daily, 30-day retention, compress after 2 days (I-012), `logs/` directory with 4 files
- **Networking**: UFW (SSH:22, Telegram/AV outbound:443, OpenAlgo localhost-only) (I-011), SSH key-only
- **Deployment**: `git pull + systemd restart` over SSH (DC-4), deploy/ directory with systemd + logrotate configs
- **Backup**: Hetzner Storage Box (~€3/mo), rsync over SSH, 30-day retention, daily ~6:15 PM IST (I-013)

**No additional questions are needed** — all infrastructure is a single VPS with well-defined services.
