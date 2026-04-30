# Espresso Network — Metrics & Monitoring Setup

> **By Cumulo** | April 2026  
> How to expose and collect Espresso Decaf validator metrics with Prometheus and visualize them in Grafana.

---

## Overview

Espresso's sequencer node exposes a Prometheus-compatible metrics endpoint out of the box when running with the `-- status` module. No additional configuration is required on the node side — metrics are available at:

```
http://<NODE_IP>:<API_PORT>/v1/status/metrics
```

This guide covers how to connect an external Prometheus instance to scrape those metrics and import the Cumulo Grafana dashboard.

---

## Prerequisites

- Espresso DA node running with the `-- status` module (included in the recommended command)
- An external Prometheus instance (any version ≥ 2.x)
- Grafana connected to that Prometheus datasource
- Network access from the Prometheus server to the node's API port

---

## Step 1 — Verify the Metrics Endpoint

From any machine with network access to the node, verify the endpoint is responding:

```bash
curl -s http://<NODE_IP>:<API_PORT>/v1/status/metrics | head -10
```

Expected output:

```
# HELP aggregator_height height
# TYPE aggregator_height gauge
aggregator_height 1
# HELP consensus_append_da_duration seconds
# TYPE consensus_append_da_duration histogram
...
```

> If the endpoint is not accessible, check that the API port is open in your firewall and that the sequencer started with the `-- status` flag.

---

## Step 2 — Add the Scrape Job to Prometheus

Edit your `prometheus.yml` and add the following scrape job:

```yaml
scrape_configs:

  - job_name: 'espresso-decaf-cumulo'
    static_configs:
      - targets: ['<NODE_IP>:<API_PORT>']
    metrics_path: '/v1/status/metrics'
    scrape_interval: 15s
```

> **Important:** The `metrics_path` field is mandatory. Without it, Prometheus defaults to `/metrics` which does not exist on Espresso nodes and the scrape will fail silently.

### Adding Multiple Nodes

To monitor multiple nodes (e.g. Decaf testnet + Mainnet), add one job per node:

```yaml
scrape_configs:

  - job_name: 'espresso-decaf-cumulo'
    static_configs:
      - targets: ['<DECAF_NODE_IP>:<PORT>']
    metrics_path: '/v1/status/metrics'
    scrape_interval: 15s

  - job_name: 'espresso-mainnet-cumulo'
    static_configs:
      - targets: ['<MAINNET_NODE_IP>:<PORT>']
    metrics_path: '/v1/status/metrics'
    scrape_interval: 15s
```

Each job will appear as a separate selectable option in the Grafana dashboard.

---

## Step 3 — Reload Prometheus

Apply the configuration without downtime:

```bash
curl -X POST http://localhost:<PROMETHEUS_PORT>/-/reload
```

Or restart the service:

```bash
sudo systemctl restart prometheus
```

---

## Step 4 — Verify the Target is Active

Check that Prometheus is successfully scraping the node:

```bash
curl -s http://localhost:<PROMETHEUS_PORT>/api/v1/targets | python3 -m json.tool | grep -A 10 "espresso"
```

A healthy target shows:

```json
"job": "espresso-decaf-cumulo",
"lastError": "",
"lastScrape": "2026-04-30T15:08:16.502Z",
"health": "up"
```

If `lastError` is not empty, check network connectivity and that the `metrics_path` is correct.

---

## Step 5 — Import the Grafana Dashboard

1. In Grafana, go to **Dashboards → Import**
2. Upload the file `espresso-grafana-dashboard.json` or paste its contents
3. Select your Prometheus datasource
4. Click **Import**

### Selecting a Node

The dashboard includes a **Node** variable at the top that filters all panels. It automatically lists all Prometheus jobs matching the pattern `espresso.*`.

- To monitor a single node: select it from the dropdown
- To compare multiple nodes: select multiple jobs or choose **All**
- New nodes added to Prometheus appear automatically without modifying the dashboard

### Dashboard Sections

| Section | Description |
|---------|-------------|
| 🟢 Node Overview | At-a-glance status cards |
| 📦 Consensus & Sync | Block height, peers, view progression, timeouts |
| 🔗 L1 Sepolia | L1 connection health and error rates |
| 🗄️ DA & Database | Data Availability scanner and PostgreSQL health |
| ⚡ Performance | Latency, throughput, and network message metrics |

---

## Firewall Considerations

The metrics endpoint is served on the same port as the Query API. If you want to restrict access:

```bash
# Allow only your Prometheus server
sudo ufw allow from <PROMETHEUS_SERVER_IP> to any port <API_PORT>

# Or restrict to localhost if Prometheus runs on the same server
sudo ufw deny <API_PORT>
sudo ufw allow from 127.0.0.1 to any port <API_PORT>
```

---

## Healthy Node — Expected Metric Values

Once the node has stake delegated and is participating in consensus, these are the expected ranges for a healthy validator:

| Metric | Healthy Value |
|--------|--------------|
| `up` | 1 |
| `consensus_libp2p_num_connected_peers` | > 5 |
| `consensus_number_of_timeouts` | Low and stable |
| `scanner_missing_blocks` | 0 |
| `scanner_missing_vid` | 0 |
| `consensus_l1_failed_requests` | 0 or very low |
| `aggregator_height` | Increasing over time |

---

*Guide maintained by [Cumulo](https://cumulo.pro) — trusted validator across the blockchain ecosystem.*
