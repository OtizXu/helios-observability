# Helios — Observability Stack

> Personal observability foundation for the Helios AI Infrastructure Lab. Built on Apple Silicon Mac mini.

[![Prometheus](https://img.shields.io/badge/Prometheus-2.55-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-11.4-F46800?logo=grafana&logoColor=white)](https://grafana.com/)
[![Loki](https://img.shields.io/badge/Loki-3.2-F9E64F?logo=grafana&logoColor=black)](https://grafana.com/oss/loki/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 🎯 What this is

This is **Stage 1 of the Helios AI Infrastructure Lab** — a hands-on personal SRE / Platform Engineering practice ground modeling a small-scale GPU cloud provider's full operational stack on commodity hardware (Apple Silicon Mac minis on a home LAN).

This stage delivers the **observability foundation**: a complete metrics + logs pipeline that subsequent stages (multi-node Kubernetes, GitOps, multi-cluster federation, chaos engineering) all build on top of.

### Architecture

```
                Mac mini #1 (192.168.x.x)
   ┌─────────────────────────────────────────────────────┐
   │  Docker Network: helios-net                          │
   │                                                      │
   │  ┌──────────────┐    ┌──────────────────────────┐   │
   │  │ Node Exporter│───→│ Prometheus               │   │
   │  │ (host        │    │ (TSDB + alert eval)      │   │
   │  │  metrics)    │    │                          │   │
   │  └──────────────┘    └──────┬───────────────────┘   │
   │                              │                       │
   │  ┌──────────────┐            │                       │
   │  │ Promtail     │───→┌───────▼──────┐               │
   │  │ (log         │    │ Loki         │               │
   │  │  collector)  │    │ (log store)  │               │
   │  └──────────────┘    └───────┬──────┘               │
   │                              │                       │
   │                      ┌───────▼──────────┐           │
   │                      │ Grafana          │           │
   │                      │ (dashboards)     │           │
   │                      └──────────────────┘           │
   │                                                      │
   │                      ┌────────────────────────────┐ │
   │  Prometheus alert ──→│ Alertmanager → Telegram    │ │
   │                      └────────────────────────────┘ │
   └──────────────────────────────────────────────────────┘
```

---

## 🛠️ Stack

| Component | Version | Purpose |
|---|---|---|
| **Prometheus** | 2.55.1 | Metrics scraping + TSDB + alert rule evaluation |
| **Node Exporter** | 1.8.2 | Host-level metrics exposition (CPU, memory, disk, network) |
| **Loki** | 3.2.1 | Log aggregation backend |
| **Promtail** | 3.2.1 | Log collector (tails files → ships to Loki) |
| **Grafana** | 11.4.0 | Web UI for metrics + logs + dashboards |
| **Alertmanager** | 0.27.0 | Alert deduplication, grouping, routing → Telegram |

All components are deployed via Docker Compose with **pinned versions** (no `:latest`) for reproducibility.

---

## 🚀 Quick Start

### Prerequisites

- macOS or Linux host with Docker 20+ and Docker Compose 2+
- 8 GB+ RAM available for the stack
- Telegram bot (for alert notifications) — optional but recommended

### Setup

```bash
git clone https://github.com/<YOUR_USERNAME>/helios-observability.git
cd helios-observability

# Create .env (do not commit!)
cp .env.example .env
# Edit .env and fill in your TELEGRAM_BOT_TOKEN and TELEGRAM_USER_ID

# Start the stack
docker compose up -d

# Verify all 6 containers are running
docker compose ps
```

### Access

| URL | Purpose | Default credentials |
|---|---|---|
| http://localhost:3000 | Grafana | `admin` / `admin` (change on first login) |
| http://localhost:9090 | Prometheus | None |
| http://localhost:9093 | Alertmanager | None |
| http://localhost:9100/metrics | Node Exporter raw metrics | None |
| http://localhost:3100/ready | Loki health endpoint | None |

---

## 📊 What you can do with this

- **Inspect host health** in Grafana with prebuilt dashboards
- **Query metrics** using PromQL at `localhost:9090`
- **Search logs** using LogQL through Grafana's Explore view
- **Receive alerts** via Telegram when host disk / CPU / memory thresholds are breached
- **Reload Prometheus config without restart**:
  ```bash
  curl -X POST http://localhost:9090/-/reload
  ```

---

## 🧠 Design Decisions

### Why pinned image versions?

Reproducibility is non-negotiable. `:latest` is convenient but makes 6-month-later debugging a nightmare. SRE first principle: deterministic infrastructure.

### Why split metrics and logs to different stores?

Metrics and logs have fundamentally different shapes. Cramming both into one DB optimizes neither:

- **Metrics** are high-cardinality time series — TSDBs (Prometheus) compress them to ~1-2 bytes per sample
- **Logs** are variable-length text — object storage with label-only indexing (Loki) is 10-100× cheaper than full-text indexing (ELK)

### Why Loki over ELK?

ELK indexes every field of every log → storage explosion at scale. Loki indexes only labels (e.g., `service=auth`) and stores log bodies as compressed chunks in object storage. Tradeoff: LogQL is grep-style, not full-text. For SRE workflows where you query by service / severity / time, Loki wins on cost.

### Why pull-based monitoring (Prometheus)?

Pull simplifies service discovery (Prometheus only needs to know endpoints), gives operational visibility for free (`up` metric tells you when targets are unreachable), and lets apps scale freely without notifying a push collector.

### Why a separate Alertmanager?

Single responsibility:
- **Prometheus** detects (evaluates rules → fires alerts)
- **Alertmanager** notifies (deduplicates, groups, routes to receivers)

This separation lets you point multiple Prometheus instances at one Alertmanager (federation pattern, used in Stage 5).

---

## 🗂️ Repository Structure

```
helios-observability/
├── docker-compose.yml          ← Orchestration root
├── prometheus/
│   ├── prometheus.yml          ← Scrape config + alert rules ref
│   └── alerts.yml              ← Alert rule definitions
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
├── grafana/
│   └── provisioning/
│       ├── datasources/        ← Prometheus + Loki auto-provisioned
│       └── dashboards/         ← Auto-imported dashboards
├── alertmanager/
│   └── alertmanager.yml
└── .env.example                ← Telegram bot template (copy → .env)
```

---

## 🚧 What's NOT in this stage

The Helios Lab is built progressively. Components added in later stages:

| Stage | Adds |
|---|---|
| 2 | Multi-node k3s Kubernetes cluster |
| 3 | GitOps (Argo CD) for app deployments |
| 4 | NVIDIA GPU Operator + Slurm-on-K8s |
| 5 | **Multi-cluster federation** with Thanos for global Prometheus view |
| 6 | Chaos engineering, Velero backup/restore drills, postmortems |
| 7 | Full IaC reproducibility (Ansible + Terraform) |

---

## 🤝 Contributing

This is a personal lab. Issues / PRs are not actively monitored, but feel free to fork for your own learning.

---

## 📜 License

MIT — see [LICENSE](LICENSE).

---

## 🙋 About

Built by Otiz Xu as part of the Helios SRE practice lab.

🔗 [LinkedIn](https://linkedin.com/in/otiz0403) · 💻 [Other projects](https://github.com/OtizXu)
