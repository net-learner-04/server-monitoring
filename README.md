# Server Monitoring Daemon

A lightweight, Discord-integrated monitoring daemon for a single Linux server (built/tested on Rocky Linux). Runs as a `systemd` service and checks system resources, services, SSL certificates, and more on independent schedules, alerting only on state changes (new issues / recoveries).

## Features

- **Resource monitoring**: CPU, memory, disk usage, swap, file descriptors, zombie processes, temperature, network throughput, disk I/O, load average, inode usage
- **Service monitoring**: checks and auto-restarts essential `systemd` services (`sshd`, `fail2ban`, `firewalld`, etc.)
- **Listening port check**: verifies expected ports (22, 443, ...) are actually open
- **SSL certificate expiry check**: warns before certificates expire
- **Security update check**: counts pending security updates (`dnf`/`yum` on RHEL-family systems)
- **Disk usage trend prediction**: estimates days until disk fills up based on recent growth
- **Reboot detection**: alerts when a server restart is detected
- **Severity levels**: WARNING / CRITICAL thresholds per metric, with different colors and `@here` mentions for critical issues
- **Recovery notifications**: sends a "resolved" alert when an issue clears, not just when it fires
- **Debounce (flapping guard)**: requires N consecutive over-threshold checks before alerting, to avoid noise from momentary spikes
- **Top-process attachment**: on critical CPU/memory alerts, includes the top resource-consuming processes
- **Weekly summary report**: average disk usage and pending security updates, sent once a week
- **Persistent state**: all state (boot time, active issues, alert history, metric history) stored in a local SQLite database (`monitor.db`)
- **Discord delivery**: retries on failure, logs a critical error if all retries fail

## Requirements

```bash
pip install schedule python-dotenv requests psutil
```

## Setup

1. Clone/copy the project to the target server.
2. Create a `.env` file in the project directory:
   ```
   DISCORD_WEBHOOK=your_discord_webhook_url
   ```
3. Adjust thresholds, service list, listening ports, and check intervals in `config.py` as needed.
4. Register the daemon as a `systemd` service (`ExecStart` should point directly at `python3 /path/to/main.py`, not a terminal multiplexer such as `tmux`):
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now server-monitor
   ```
5. Verify it's running:
   ```bash
   sudo systemctl status server-monitor
   sudo journalctl -u server-monitor -f
   ```

No cron jobs are needed — the daemon schedules its own checks internally and runs continuously.

## File Overview

| File | Purpose |
|---|---|
| `main.py` | Daemon entry point; schedules all checks and handles graceful shutdown |
| `check.py` | All system/service/security check functions |
| `alert.py` | Discord formatting/sending, boot-time tracking, weekly summary |
| `config.py` | Thresholds, service list, ports, intervals, and other settings |
| `state_db.py` | SQLite-backed state, alert history, and metric history storage |
| `logger.py` | Rotating file logger setup |

## Notes

- `psutil.net_connections()` (used for the port check) may need root privileges to see all sockets, depending on which user runs the service.
- Security update checks assume a `dnf`/`yum`-based distro (RHEL family); on other distros the check is skipped without error.
- Debounce counters are held in memory, so a daemon restart resets them — an ongoing issue will simply re-trigger the debounce count on the next few checks.
