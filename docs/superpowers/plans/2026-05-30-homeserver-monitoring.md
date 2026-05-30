# Home Server Monitoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a one-command Docker stack that gives a Mac mini (Ubuntu) home server a live visual dashboard for system health, temperatures, Docker containers, and internet speed, with Telegram alerts and secure remote access.

**Architecture:** Netdata (host-networked container) provides the dashboard and alert engine, reading host metrics via mounted `/proc`, `/sys`, and the Docker socket. A `speedtest-exporter` container exposes internet-speed metrics that Netdata scrapes every 6 hours. Tailscale is installed on the host for secure remote access. All Netdata config lives in a version-controlled `./netdata/` directory; secrets are injected from a git-ignored `.env` via the container environment and never written into config files.

**Tech Stack:** Docker + Docker Compose, Netdata (stable), `ghcr.io/miguelndecarvalho/speedtest-exporter`, Tailscale, lm-sensors (host), Telegram Bot API.

---

## File Structure

| File | Responsibility |
|---|---|
| `docker-compose.yml` | Defines the `netdata` and `speedtest-exporter` services and named volumes. |
| `.env.example` | Documents required secrets (Telegram token + chat ID). Copied to `.env` by the user. |
| `.gitignore` | Ensures `.env` and runtime data never get committed. |
| `netdata/health_alarm_notify.conf` | Enables Telegram alerts; references secrets from container env. |
| `netdata/go.d/prometheus.conf` | Tells Netdata to scrape the speedtest-exporter every 6 hours. |
| `netdata/health.d/temperature.conf` | Temperature alert thresholds (warn/crit). |
| `README.md` | Step-by-step host prep (Docker, lm-sensors, Tailscale), Telegram bot/chat-ID setup, bring-up, verification checklist, and PostgreSQL future-enable note. |

> Note: Netdata reads user config from `/etc/netdata` and falls back to stock defaults for anything not present, so the small `./netdata/` directory only needs to contain our overrides. The default Netdata health alerts already cover CPU load, RAM, disk space, and container (cgroup) health — we add Telegram delivery and a temperature alert on top.

---

### Task 1: Repository scaffolding

**Files:**
- Modify: `.gitignore`
- Create: `.env.example`
- Create: `netdata/.gitkeep`, `netdata/go.d/.gitkeep`, `netdata/health.d/.gitkeep`

- [ ] **Step 1: Ensure `.gitignore` ignores secrets and runtime data**

Write `.gitignore` (overwrite) with exactly:

```gitignore
# Secrets — never commit
.env

# Local runtime / editor noise
*.local
data/
```

- [ ] **Step 2: Create `.env.example`**

Create `.env.example` with exactly:

```dotenv
# Telegram bot token from @BotFather (see README step "Create the Telegram bot")
TELEGRAM_BOT_TOKEN=123456789:AAExampleReplaceMeWithYourRealToken

# Numeric Telegram chat ID the alerts are sent to (see README step "Get your chat ID")
TELEGRAM_CHAT_ID=123456789
```

- [ ] **Step 3: Create the Netdata config directory skeleton**

Run:

```bash
mkdir -p netdata/go.d netdata/health.d
touch netdata/.gitkeep netdata/go.d/.gitkeep netdata/health.d/.gitkeep
```

(The `.gitkeep` files keep the directories tracked; they are replaced by real config in later tasks but harmless if left.)

- [ ] **Step 4: Commit**

```bash
git add .gitignore .env.example netdata/.gitkeep netdata/go.d/.gitkeep netdata/health.d/.gitkeep
git commit -m "chore: scaffold monitoring repo structure and secrets template"
```

---

### Task 2: Docker Compose stack

**Files:**
- Create: `docker-compose.yml`

- [ ] **Step 1: Write `docker-compose.yml`**

Create `docker-compose.yml` with exactly:

```yaml
services:
  netdata:
    image: netdata/netdata:stable
    container_name: netdata
    hostname: homeserver
    restart: unless-stopped
    pid: host
    network_mode: host
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    env_file:
      - .env
    volumes:
      - ./netdata:/etc/netdata
      - netdatalib:/var/lib/netdata
      - netdatacache:/var/cache/netdata
      - /:/host/root:ro,rslave
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
      - /var/log:/host/var/log:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro

  speedtest-exporter:
    image: ghcr.io/miguelndecarvalho/speedtest-exporter:latest
    container_name: speedtest-exporter
    restart: unless-stopped
    ports:
      - "9798:9798"

volumes:
  netdatalib:
  netdatacache:
```

Why these settings:
- `network_mode: host` exposes the dashboard on the host's `:19999` and lets Netdata reach the exporter at `127.0.0.1:9798`.
- `pid: host`, the `cap_add`, the `security_opt`, and the read-only host mounts are Netdata's documented requirements for full system + temperature + Docker visibility.
- `./netdata:/etc/netdata` makes all our config version-controlled; `env_file: .env` injects the Telegram secrets into the container environment.

- [ ] **Step 2: Validate compose syntax**

Run (on any machine with Docker; if Docker is not installed locally, defer this to the server after Task 6):

```bash
cp .env.example .env   # placeholder values are fine just for validation
docker compose config >/dev/null && echo "compose OK"
```

Expected: prints `compose OK` with no error. (Delete the placeholder `.env` afterward if validating locally: `rm .env`.)

- [ ] **Step 3: Commit**

```bash
git add docker-compose.yml
git commit -m "feat: add docker compose stack for netdata + speedtest exporter"
```

---

### Task 3: Telegram alert delivery

**Files:**
- Create: `netdata/health_alarm_notify.conf`

- [ ] **Step 1: Write the notification config**

Create `netdata/health_alarm_notify.conf` with exactly:

```bash
#!/usr/bin/env bash
# Netdata alert notifications -> Telegram.
# Netdata sources the stock config first, then this file (overriding it).
# Secrets are pulled from the container environment (see .env / docker-compose
# env_file) so no token is ever stored in this committed file.

SEND_TELEGRAM="YES"
TELEGRAM_BOT_TOKEN="${TELEGRAM_BOT_TOKEN}"
DEFAULT_RECIPIENT_TELEGRAM="${TELEGRAM_CHAT_ID}"
```

Why this works: `health_alarm_notify.conf` is sourced as a bash script by Netdata's `alarm-notify.sh`, which runs inside the container where `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` are present (injected via `env_file`). The variable references expand at notify time; the secrets stay only in `.env`.

- [ ] **Step 2: Commit**

```bash
git add netdata/health_alarm_notify.conf
git commit -m "feat: route netdata alerts to telegram via env-injected secrets"
```

(End-to-end verification of a real alert happens in the README checklist on the server — Task 6, verification step.)

---

### Task 4: Internet speed scraping (every 6 hours)

**Files:**
- Create: `netdata/go.d/prometheus.conf`

- [ ] **Step 1: Write the scrape job**

Create `netdata/go.d/prometheus.conf` with exactly:

```yaml
# Scrape the speedtest-exporter. Each scrape RUNS a real speed test (~30-60s),
# so we scrape only every 6 hours and allow a long timeout.
jobs:
  - name: speedtest
    url: http://127.0.0.1:9798/metrics
    timeout: 90
    update_every: 21600   # 6 hours, in seconds
```

Why: Netdata's generic `prometheus` go.d collector turns any Prometheus endpoint into charts. Because the exporter performs the test on each scrape, the 6-hour `update_every` is exactly the test cadence. `timeout: 90` prevents the long-running test from being cut off (the exporter's default per-test budget is well under this).

- [ ] **Step 2: Commit**

```bash
git add netdata/go.d/prometheus.conf
git commit -m "feat: scrape internet speed every 6h via speedtest exporter"
```

---

### Task 5: Temperature alert thresholds

**Files:**
- Create: `netdata/health.d/temperature.conf`

- [ ] **Step 1: Write the temperature alarm**

Create `netdata/health.d/temperature.conf` with exactly:

```text
# Warn/crit on high sensor temperature.
# IMPORTANT: the `on:` value below targets the chart CONTEXT. On most systems
# this is `sensors.temperature`. If no alert appears after setup, open the
# dashboard, find your temperature chart, read its context from the chart's
# info/"i" menu, and replace the value below to match, then restart netdata.
 template: sensor_temperature_high
       on: sensors.temperature
    class: Utilization
     type: System
component: Sensors
   lookup: max -1m
    units: Celsius
    every: 60s
     warn: $this > 75
     crit: $this > 90
    delay: down 5m
     info: max sensor temperature over the last minute
       to: sysadmin
```

Why `max` over `-1m`: a single hot core should trip the alert even if the average looks fine. `sysadmin` is Netdata's default role and routes to the Telegram recipient configured in Task 3.

- [ ] **Step 2: Commit**

```bash
git add netdata/health.d/temperature.conf
git commit -m "feat: add high temperature warn/crit alert"
```

---

### Task 6: README — host prep, setup, verification

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write the README**

Create `README.md` with exactly:

````markdown
# Home Server Monitoring

A one-command Docker stack that gives this Ubuntu home server (an old Mac mini) a
live dashboard for system health, temperatures, Docker containers, and internet
speed — with Telegram alerts and secure remote access via Tailscale.

Dashboard: **http://<server-ip>:19999** (or over Tailscale from anywhere).

## What runs

- **Netdata** — the dashboard + alert engine (CPU, RAM, disk, network, temps, Docker).
- **speedtest-exporter** — internet speed, scraped by Netdata every 6 hours.
- **Tailscale** — installed on the host (not a container) for secure remote access.

---

## 1. Host prerequisites

### Docker + Compose
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"   # log out/in afterwards so docker runs without sudo
```

### Temperature sensors (required for temps to show up)
```bash
sudo apt update
sudo apt install -y lm-sensors
sudo sensors-detect --auto
# Apple hardware exposes temps/fans via these modules:
sudo modprobe applesmc || true
sudo modprobe coretemp || true
sensors    # confirm you see temperatures here before continuing
```
If `sensors` prints temperatures, Netdata will pick them up automatically.

---

## 2. Create the Telegram bot

1. In Telegram, message **@BotFather**, send `/newbot`, follow the prompts.
2. Copy the **bot token** it gives you (looks like `123456789:AA...`).

### Get your chat ID
1. Send any message to your new bot.
2. Run (replace `<TOKEN>`):
   ```bash
   curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates" | grep -o '"chat":{"id":[0-9-]*'
   ```
   The number after `"id":` is your **chat ID**.

---

## 3. Configure secrets

```bash
cp .env.example .env
```
Edit `.env` and set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` to the values above.
`.env` is git-ignored and never committed.

---

## 4. Bring up the stack

```bash
docker compose up -d
docker compose ps      # both containers should be "running"/"healthy"
```

Open **http://<server-ip>:19999**.

---

## 5. Remote access with Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
Follow the printed login URL. Afterwards, reach the dashboard from any device on
your tailnet at **http://<tailscale-ip>:19999** — no public ports opened.

---

## 6. Verification checklist

- [ ] Dashboard loads at `http://<server-ip>:19999`.
- [ ] **Temperatures** appear (search the dashboard for "sensors"/"temperature").
- [ ] Each running **Docker container** shows up with live CPU/RAM/IO stats.
- [ ] After the first 6-hour scrape (or force one — see below), an **internet
      speed** chart appears under the Prometheus/speedtest section.
- [ ] A **test alert reaches Telegram**:
      ```bash
      docker exec -it netdata /usr/libexec/netdata/plugins.d/alarm-notify.sh test
      ```
      You should receive a test message in your Telegram chat.
- [ ] Dashboard reachable over **Tailscale** from a device off your home network.

### Force an immediate speed test (optional)
```bash
curl -s http://localhost:9798/metrics | grep speedtest_
```
This triggers a test now and prints the raw metrics.

---

## Future: monitor PostgreSQL

When you add a self-hosted PostgreSQL container, enable Netdata's Postgres
collector by creating `netdata/go.d/postgres.conf`:

```yaml
jobs:
  - name: local
    dsn: 'postgres://<user>:<password>@127.0.0.1:5432/postgres?sslmode=disable'
```
Put the credentials in `.env` and reference them, or use a read-only monitoring
role. Then `docker compose restart netdata`. Postgres charts will appear
automatically. (Not enabled yet — no database exists.)

---

## Updating

```bash
docker compose pull && docker compose up -d
```
````

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add setup, host prep, and verification README"
```

---

### Task 7: Final stack validation (on the server)

This task runs on the Mac mini after the repo is present and `.env` is filled in.

- [ ] **Step 1: Validate compose and bring up**

```bash
docker compose config >/dev/null && echo "compose OK"
docker compose up -d
docker compose ps
```
Expected: `compose OK`, then both `netdata` and `speedtest-exporter` listed as running.

- [ ] **Step 2: Confirm the dashboard and metrics**

Open `http://<server-ip>:19999`. Walk the README **Verification checklist** end to
end (temps, containers, speed test, Telegram test alert, Tailscale access).

- [ ] **Step 3: Tag the working setup**

```bash
git tag -a v1.0 -m "Working monitoring stack: netdata + speedtest + telegram + tailscale"
```

---

## Notes for the implementer

- This is an infrastructure/config project, not application code, so "tests" are
  the validation commands (`docker compose config`) and the README verification
  checklist run against real hardware. There is no unit-test suite.
- Never write the Telegram token into any committed file — it lives only in `.env`.
- If temperature alerts never fire, the chart context in
  `netdata/health.d/temperature.conf` likely differs from `sensors.temperature`;
  read the actual context from the dashboard and update it (noted inline in that file).
