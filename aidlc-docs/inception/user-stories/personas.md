# User Personas for Auto-Trader (AT)

## Persona 1: Trader/Owner

| Attribute | Detail |
|---|---|
| **Role** | Primary user — owns the trading capital and makes all strategic decisions |
| **Goals** | Grow capital through disciplined, automated swing trading on BSE; maintain full visibility into system decisions; retain override authority in Trial mode |
| **Responsibilities** | Configure seed watchlist and candidate pool; review daily Telegram reports; approve/reject watchlist changes (Sandbox/Trial); approve/reject trade proposals (Trial); promote between modes (Sandbox → Trial → Production); review backtest results |
| **Interactions** | Telegram bot (daily reports, heartbeats, alerts, HITL approvals), configuration files (watchlist, candidate pool, guardrail parameters), CLI (ad-hoc queries, mode promotion) |
| **Pain points** | Missed Telegram alerts during busy hours; unclear why a trade was blocked; difficulty reconstructing past decisions |
| **Technical level** | Comfortable with Python and CLIs but not a full-time developer |

## Persona 2: System Administrator

| Attribute | Detail |
|---|---|
| **Role** | Deploys, maintains, and troubleshoots the system on the Hetzner VPS |
| **Goals** | Keep the system running 24x7 with minimal manual intervention; ensure security, backups, and observability; quickly diagnose issues via structured per-module logs |
| **Responsibilities** | Provision and harden VPS; deploy code (git pull + systemd restart); monitor per-module log files for errors and system health; handle broker IP whitelisting; manage backups and log rotation; respond to critical system alerts |
| **Interactions** | SSH to VPS, systemd service management, per-module log files (data, model, execution, scheduler), Telegram (critical system alerts only) |
| **Pain points** | Unplanned VPS reboots during market hours; all logs in a single file making diagnosis hard; log noise obscuring real issues; credential rotation complexity |
| **Technical level** | Strong Linux sysadmin and Python environment management skills |

> **Note**: In practice, these two personas may be the same person. They are separated to clarify distinct responsibilities and interaction patterns.
