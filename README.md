# Tycho Simulator Risk Monitoring

Polls a [Tycho Simulator](https://github.com/propeller-heads/tycho-simulation) API to evaluate swap execution risk across Ethereum DEX pools, persists results in Postgres, fires configurable alerts, and exposes a REST API with Prometheus metrics.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      main.rs                            │
│                                                         │
│  ┌──────────────────┐       ┌────────────────────────┐  │
│  │ Polling Loop      │       │ Axum REST API          │  │
│  │ (tokio::spawn)    │       │ (tokio::spawn)         │  │
│  │                   │       │                        │  │
│  │ POST /simulate ──►│       │ GET /health            │  │
│  │   + retry w/      │       │ GET /metrics           │  │
│  │     exp. backoff  │       │ GET /api/v1/results    │  │
│  │                   │       │ GET /api/v1/pools      │  │
│  │ ┌───────────────┐ │       │ GET /api/v1/pools/     │  │
│  │ │ Alert Engine   │ │       │     high-risk          │  │
│  │ │ risk_score     │ │       │                        │  │
│  │ │ slippage_bps   │ │       │ Rate Limit + API Key   │  │
│  │ │ webhook POST   │ │       │ middleware             │  │
│  │ └───────────────┘ │       └────────────────────────┘  │
│  └────────┬─────────┘                    │               │
│           │                              │               │
│           ▼                              ▼               │
│  ┌──────────────────────────────────────────────────┐    │
│  │              PostgreSQL (sqlx)                    │    │
│  │  result ──< pool_result                           │    │
│  └──────────────────────────────────────────────────┘    │
│           │                                              │
│           ▼                                              │
│  ┌────────────────┐                                      │
│  │  Prometheus     │                                     │
│  │  /metrics       │                                     │
│  └────────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

## Module Overview

| Module | Responsibility |
|--------|---------------|
| `config.rs` | JSON config loading with env-var overrides |
| `client.rs` | Simulation HTTP client with retry + exponential backoff |
| `db.rs` | Postgres connection pool, migrations, transactional inserts |
| `api.rs` | Axum REST API, healthcheck, rate limiting, API key auth |
| `alerts.rs` | Threshold evaluation + webhook delivery |
| `metrics.rs` | Prometheus counters, histograms, gauges |
| `models.rs` | Request/response/DB row/API response types |
| `errors.rs` | `thiserror`-based error enum |

## Prerequisites

- Rust 1.75+
- PostgreSQL 14+

## Quick Start

```bash
# 1. Start Postgres (example with Docker)
docker run -d --name pg \
  -e POSTGRES_USER=rust \
  -e POSTGRES_PASSWORD=teste \
  -e POSTGRES_DB=postgres \
  -p 5432:5432 postgres:16

# 2. Configure
cp config.json config.local.json
# Edit config.local.json with your values

# 3. Run
CONFIG_PATH=config.local.json RUSTFLAGS="-C target-cpu=native -C link-arg=-s" cargo run --release > system-monitoring.log 2>&1 &
```

Tables and indexes are created automatically on startup via `migrations/001_create_tables.sql`.

## Configuration

All settings are in `config.json`. Secrets can be overridden with environment variables.

| Field | Env Override | Default | Description |
|-------|-------------|---------|-------------|
| `database_url` | `DATABASE_URL` | — | Postgres connection string |
| `simulation_api_url` | `SIMULATION_API_URL` | — | Tycho simulator POST endpoint |
| `poll_interval_secs` | — | `60` | Seconds between cycles. `0` = single-shot |
| `api_port` | `API_PORT` | `3000` | REST API listen port |
| `api_key` | `API_KEY` | `null` | If set, `/api/v1/*` routes require `X-API-Key` header |
| `rate_limit_rps` | — | `10` | Max requests/second across all API endpoints |
| `retry.max_retries` | — | `3` | Retry attempts on 429/5xx |
| `retry.initial_backoff_ms` | — | `500` | Initial retry backoff (doubles each attempt) |
| `alerts.risk_score_threshold` | — | `70` | Pools with `risk_score >=` this fire an alert |
| `alerts.slippage_bps_threshold` | — | `500` | Slippage in bps `>=` this fires an alert |
| `alerts.webhook_url` | — | `null` | POST JSON alerts to this URL |
| `token_pairs` | — | — | Array of `{label, token_in, token_out, amounts}` |

Example:

```json
{
    "database_url": "postgres://user:pass@localhost:5432/tycho_risk",
    "simulation_api_url": "http://simulator:3000/simulate",
    "poll_interval_secs": 60,
    "api_port": 3000,
    "api_key": "my-secret-key",
    "rate_limit_rps": 20,
    "retry": {
        "max_retries": 3,
        "initial_backoff_ms": 500
    },
    "alerts": {
        "risk_score_threshold": 70,
        "slippage_bps_threshold": 500,
        "webhook_url": "https://hooks.slack.com/services/..."
    },
    "token_pairs": [
        {
            "label": "DAI → USDC",
            "token_in": "0x6b175474e89094c44da98b954eedeac495271d0f",
            "token_out": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
            "amounts": ["1000000000000000000"]
        }
    ]
}
```

## API Endpoints

### Public

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Returns status, version, and DB connectivity check |
| `GET` | `/metrics` | Prometheus scrape endpoint |

### Protected (require `X-API-Key` header if `api_key` is configured)

| Method | Path | Query Params | Description |
|--------|------|-------------|-------------|
| `GET` | `/api/v1/results` | `limit`, `offset`, `min_risk_score` | List simulation results |
| `GET` | `/api/v1/results/{id}` | — | Get single result by UUID |
| `GET` | `/api/v1/results/{id}/pools` | — | Pool results for a simulation |
| `GET` | `/api/v1/pools` | `limit`, `offset`, `pool_address`, `min_risk_score` | List pool results |
| `GET` | `/api/v1/pools/high-risk` | — | Pools above configured `risk_score_threshold` |

### Health Response

```json
{
    "status": "ok",
    "version": "0.2.0",
    "database": "connected"
}
```

`status` is `"degraded"` if the database probe fails.

## Prometheus Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `simulation_requests_total` | Counter | `pair`, `status` | Total simulation calls |
| `simulation_duration_ms` | Histogram | `pair` | Round-trip time per call |
| `pool_risk_score` | Histogram | `pool` | Risk scores observed |
| `pool_slippage_bps` | Histogram | `pool` | Slippage in basis points |
| `pool_gas_used` | Histogram | `pool` | Gas consumption |
| `pool_utilization_bps` | Histogram | `pool` | Pool utilization |
| `latest_block_number` | Gauge | — | Last block polled |
| `latest_matching_pools` | Gauge | — | Matching pools in last cycle |
| `latest_candidate_pools` | Gauge | — | Candidate pools in last cycle |
| `alerts_fired_total` | Counter | `type` | Alert events |

## Alert System

Each simulation cycle evaluates every pool against two thresholds:

- **`high_risk_score`** — `execution_risk.risk_score >= risk_score_threshold`
- **`high_slippage`** — any `slippage_bps[i] >= slippage_bps_threshold`

Fired alerts are:
1. Logged at `WARN` level
2. Counted in `alerts_fired_total` Prometheus metric
3. POSTed as JSON to `webhook_url` (if configured)

Alert payload:

```json
{
    "alert_type": "high_risk_score",
    "severity": "critical",
    "pool_address": "0x...",
    "pool_name": "SketchyDEX ETH/SCAM",
    "message": "Risk score 92 >= threshold 70 on pool 0x...",
    "value": 92,
    "threshold": 70,
    "block_number": 19500001,
    "request_id": "uuid-v4"
}
```

## Monitoring (Prometheus + Grafana)

Both services are assumed to be installed directly on the machine. The `monitoring/` directory contains all required config files.

### File layout

```
monitoring/
├── prometheus.yml                          # Scrape config  → copy to Prometheus config dir
└── grafana/
    ├── provisioning/
    │   ├── datasources/prometheus.yml      # Datasource     → copy to Grafana provisioning dir
    │   └── dashboards/dashboard.yml        # Dashboard loader → copy to Grafana provisioning dir
    └── dashboards/
        └── tycho-risk-monitor.json         # Pre-built dashboard → copy to Grafana dashboards dir
```

---

### macOS (Homebrew)

#### Install

```bash
brew install prometheus grafana
```

#### Configure Prometheus

```bash
cp monitoring/prometheus.yml /opt/homebrew/etc/prometheus.yml
```

#### Configure Grafana

```bash
# Datasource
cp monitoring/grafana/provisioning/datasources/prometheus.yml /opt/homebrew/etc/grafana/provisioning/datasources/prometheus.yml

# Dashboard provider
cp monitoring/grafana/provisioning/dashboards/dashboard.yml \
   /opt/homebrew/etc/grafana/provisioning/dashboards/dashboard.yml

# Dashboard JSON — Grafana loads from the path set in dashboard.yml
sudo mkdir -p /var/lib/grafana/dashboards
sudo cp monitoring/grafana/dashboards/tycho-risk-monitor.json \
        /var/lib/grafana/dashboards/tycho-risk-monitor.json
```

#### Start / restart services

```bash
brew services start prometheus
brew services start grafana

# Or restart if already running
brew services restart prometheus
brew services restart grafana
```

---

### Linux (apt / systemd)

#### Install on Linux

```bash
# Prometheus
sudo apt-get install -y prometheus

# Grafana
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update && sudo apt-get install -y grafana
```

#### Configure Prometheus on Linux

```bash
sudo cp monitoring/prometheus.yml /etc/prometheus/prometheus.yml
```

#### Configure Grafana on Linux

```bash
# Datasource
sudo cp monitoring/grafana/provisioning/datasources/prometheus.yml \
        /etc/grafana/provisioning/datasources/prometheus.yml

# Dashboard provider
sudo cp monitoring/grafana/provisioning/dashboards/dashboard.yml \
        /etc/grafana/provisioning/dashboards/dashboard.yml

# Dashboard JSON
sudo mkdir -p /var/lib/grafana/dashboards
sudo cp monitoring/grafana/dashboards/tycho-risk-monitor.json \
        /var/lib/grafana/dashboards/tycho-risk-monitor.json
```

#### Start / restart services on Linux

```bash
sudo systemctl enable --now prometheus
sudo systemctl enable --now grafana-server

# Or restart if already running
sudo systemctl restart prometheus
sudo systemctl restart grafana-server
```

---

### Access

| Service    | URL                   | Default credentials |
| ---------- | --------------------- | ------------------- |
| Prometheus | http://localhost:9090 | —                   |
| Grafana    | http://localhost:3000 | admin / admin       |

The dashboard **Tycho Risk Monitor** loads automatically on first Grafana start.

> **Note:** Grafana defaults to port 3000, the same as the app. If both run on the same machine, change the Grafana port in its config:
>
> - macOS: edit `/opt/homebrew/etc/grafana/grafana.ini` → `[server] http_port = 3001`
> - Linux: edit `/etc/grafana/grafana.ini` → `[server] http_port = 3001`
>
> Then restart Grafana.

---

### Change the app port

If the app runs on a port other than `3000`, update the scrape target in `monitoring/prometheus.yml` and recopy it:

```yaml
static_configs:
  - targets:
      - localhost:3000   # ← change port here
```

### Reload Prometheus config without restart

```bash
curl -X POST http://localhost:9090/-/reload
```

### Dashboard panels

| Panel                      | Metric(s)                                          | Description                       |
| -------------------------- | -------------------------------------------------- | --------------------------------- |
| Latest Block               | `latest_block_number`                              | Last block seen                   |
| Matching / Candidate Pools | `latest_matching_pools`, `latest_candidate_pools`  | Last cycle pool counts            |
| Alerts Fired (5m)          | `alerts_fired_total`                               | Alert spike indicator             |
| Simulation Request Rate    | `simulation_requests_total`                        | req/s by pair and status          |
| Simulation Duration        | `simulation_duration_ms`                           | p50 / p95 / p99 by pair           |
| Pool Risk Score            | `pool_risk_score`                                  | p95 per pool — red at ≥ 70        |
| Pool Slippage              | `pool_slippage_bps`                                | p95 per pool — red at ≥ 500 bps   |
| Pool Gas Used              | `pool_gas_used`                                    | p95 per pool                      |
| Pool Utilization           | `pool_utilization_bps`                             | p95 per pool                      |
| Alerts by Type             | `alerts_fired_total`                               | Rate and totals per alert type    |

## Docker

```bash
# Build
docker build -t tycho-risk-monitor .

# Run
docker run -d \
  -e DATABASE_URL=postgres://user:pass@host:5432/db \
  -e SIMULATION_API_URL=http://simulator:3000/simulate \
  -e API_KEY=my-secret \
  -p 3000:3000 \
  tycho-risk-monitor
```

Override `config.json` by mounting a volume:

```bash
docker run -v ./my-config.json:/etc/app/config.json ...
```

## Development

```bash
# Check
cargo check

# Lint
cargo clippy --all-targets --all-features

# Test
cargo test

# Build release
RUSTFLAGS="-C target-cpu=native -C link-arg=-s" cargo build --release
```

## CI

GitHub Actions pipeline (`.github/workflows/ci.yml`) runs on push/PR to `main`:

1. `cargo fmt --check`
2. `cargo clippy` (warnings = errors)
3. `cargo test`
4. Release build
5. Docker build (main branch only)

## Database Schema

Two tables with a one-to-many relationship:

- **`result`** — one row per simulation API call (request_id, block_number, meta)
- **`pool_result`** — one row per DEX pool in the response (risk_score, slippage_bps, gas_used, etc.)

Migrations run automatically at startup. All indexes use `IF NOT EXISTS` for idempotency.

## License

MIT
