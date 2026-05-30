# Grafana Monitoring Stack — Design

**Date:** 2026-05-30
**Status:** Approved

## Goal

Add a second, visually richer monitoring stack to the home server (Mac mini,
Ubuntu) alongside the existing Netdata stack, built on Grafana + Prometheus +
Loki. It provides two dashboards — a system overview and a centralized
per-container logs view — and sends log-aware alerts to Telegram. Netdata
continues to run unchanged.

## Approach

Assemble proven, free, open-source tools as a self-contained docker-compose
stack in a new `grafana-stack/` directory inside the existing
`homeserver_monitoring` repo. The stack is fully independent of the Netdata
stack (separate compose project, separate `.env`, non-conflicting ports), so
either can be started, stopped, or updated without affecting the other.

Everything is provisioned as code (datasources, dashboards, alert rules), so a
single `docker compose up -d` yields a working setup with no manual clicking.

Rejected alternative: extending Netdata. The user explicitly wants Grafana's
customizable dashboards and centralized log view, which Netdata does not provide.

## Components

All ports below are distinct from the Netdata stack (`:19999`, `:9798`).

| Service | Port | Role |
|---|---|---|
| **Grafana** | 3000 | UI: the two dashboards and the alerting engine. |
| **Prometheus** | 9090 | Time-series metrics store; scrapes the exporters below. |
| **Loki** | 3100 | Log store. |
| **Grafana Alloy** | 12345 | Log collector. Auto-discovers all running Docker containers via the Docker socket and ships their logs to Loki, labeled by `container` name. |
| **node_exporter** | 9100 | Host metrics: CPU, per-core usage, temperature (hwmon), disk, RAM, network. |
| **cAdvisor** | 8080 | Per-container metrics: CPU, memory, network, IO. |

**Host prerequisite:** enable the Docker daemon's own Prometheus metrics endpoint
by adding `"metrics-addr": "127.0.0.1:9323"` (and `"experimental": true`) to
`/etc/docker/daemon.json` and restarting Docker. This exposes
`engine_daemon_container_states_containers{state=...}` and container action
counters — the clean source for "containers running / stopped / restart (crash)
counts."

## Data Flow

1. node_exporter, cAdvisor, and the Docker daemon expose Prometheus metrics.
   Prometheus scrapes all three (and itself) on a regular interval and stores them.
2. Grafana Alloy watches the Docker socket, discovers every container, and
   streams each container's logs to Loki with a `container` label.
3. Grafana queries Prometheus (metrics) and Loki (logs) as provisioned
   datasources and renders the two dashboards.
4. Grafana's alerting engine evaluates provisioned rules against Prometheus and
   Loki, and sends notifications to Telegram via the existing bot.

## Dashboards

### Dashboard 1 — System Overview (main/home dashboard)
Provisioned community dashboards plus one custom panel row:
- **Node Exporter Full** (Grafana.com dashboard ID 1860): CPU usage, per-core
  usage, temperature, disk space, RAM, network, uptime. Temperature panels are
  expected to surface `node_hwmon_temp_celsius`; the real CPU temps come from the
  `coretemp` chip (the `applesmc` chip reports some bogus values, as found during
  the Netdata work).
- **cAdvisor / Docker** (community dashboard, e.g. ID 14282): per-container CPU,
  memory, network, IO.
- A small **custom top row**: "containers running / stopped / restart counts"
  using the Docker daemon metrics
  (`engine_daemon_container_states_containers`, `engine_daemon_container_actions_seconds_count`).

Grafana's default home dashboard is set (via provisioning) to the System Overview.

### Dashboard 2 — Logs
A custom Grafana dashboard backed by Loki:
- A `container` dashboard template variable populated from Loki label values
  (`label_values(container)`), defaulting to "All".
- A **Logs panel set to repeat by the `container` variable**, so Grafana renders
  one separate logs panel per container automatically. New containers (e.g. a
  future PostgreSQL) appear as their own panel with no config change.
- Each panel supports text search/filtering and time-range scoping.

## Alerting (Grafana → Telegram)

Provisioned as code (contact point, notification policy, alert rules). Reuses the
existing Telegram bot via `TG_BOT_TOKEN` / `TG_CHAT_ID` (same values as the
Netdata `.env`, copied into this stack's `.env`).

Starter rules, deliberately focused on gaps Netdata does not cover well (to avoid
duplicate pings):
- **Log-based:** spike in `error` / `panic` / `fatal` log lines across containers
  (a Loki query rule) — the headline capability Netdata lacks.
- **Container down / crash-loop:** a container expected to be running is absent,
  or its restart count climbs rapidly (Docker daemon metrics).
- **Disk near-full** and **high CPU temperature** are included but can be left to
  Netdata; thresholds are tunable.

## Deliverables

Inside `grafana-stack/`:
- `docker-compose.yml` — the six services above with correct mounts/networks.
- `.env.example` — Grafana admin password + `TG_BOT_TOKEN` / `TG_CHAT_ID`.
- `prometheus/prometheus.yml` — scrape configs for node_exporter, cAdvisor,
  Docker daemon, and Prometheus itself.
- `loki/loki-config.yml` — single-binary Loki config (filesystem storage).
- `alloy/config.alloy` — Docker discovery + log shipping to Loki.
- `grafana/provisioning/datasources/*.yml` — Prometheus + Loki datasources.
- `grafana/provisioning/dashboards/*.yml` + dashboard JSON files — the two
  dashboards (community JSON for system/cAdvisor, custom JSON for logs and the
  container-state row), with System Overview set as the home dashboard.
- `grafana/provisioning/alerting/*.yml` — contact point, policy, alert rules.

Repo-level:
- Update `.gitignore` to ignore `grafana-stack/.env` and Grafana/Prometheus/Loki
  runtime data dirs.
- A new section in the top-level `README.md` (or a `grafana-stack/README.md`)
  covering the Docker-metrics host step, bring-up, and the LAN/Tailscale URLs.

## Scope Boundaries (YAGNI)

- Netdata stack is not modified.
- No external object storage for Loki/Prometheus — local filesystem with modest
  retention is sufficient for a single home server.
- No Grafana user management beyond the admin account.
- PostgreSQL is not pre-wired; it will simply appear in logs (auto-discovered)
  and can later be scraped by Prometheus when added.

## Verification

- `docker compose up -d` brings all six services to running/healthy.
- Grafana reachable at `http://<server-ip>:3000` (and via Tailscale), logs in
  with the admin credentials, lands on the System Overview dashboard.
- System Overview shows live CPU/per-core/temperature/disk/RAM and per-container
  metrics; the container-state row shows running count and restart counts.
- Logs dashboard shows one panel per running container with live log lines.
- A deliberately triggered test (e.g. emitting error log lines, or stopping a
  test container) produces a Telegram alert.
- Netdata (`:19999`) continues to operate unaffected.
