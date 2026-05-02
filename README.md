# cumulo-espresso-infra

> Infrastructure documentation for Cumulo's Espresso Network validator node - hardware setup, DA node installation, configuration, monitoring, and operations runbook.

---

## 🌐 About Espresso Network

[Espresso Network](https://espressosys.com) is a decentralized confirmation layer for Ethereum rollups. It uses the **HotShot** consensus protocol to confirm blocks in a fast, trust-minimized way without relying on a centralized sequencer.

**Decaf** is Espresso's persistent long-running testnet, launched in September 2024 and upgraded to Proof-of-Stake (PoS) with HotShot in April 2025. The top 100 validators by stake participate in consensus.

---

## 📁 Repository Structure

```
cumulo-espresso-infra/
├── README.md                          # This file
├── docs/
│   ├── validator-setup.md             # Full DA node setup guide
│   ├── node-commands.md               # Day-to-day operations cheatsheet
│   └── metrics-setup.md              # Prometheus + Grafana monitoring
├── monitoring/
│   └── prometheus-scrape-config.yml   # Prometheus scrape job for Espresso metrics
└── keys/
    └── README.md                      # Key management notes (no private keys stored here)
```

---

## 🖥️ Hardware

| Component | Specification |
|-----------|--------------|
| CPU | 6 cores (4 sequencer + 2 PostgreSQL) |
| RAM | 12 GB (8 GB + 4 GB) |
| Disk | 100 GB SSD (DA node with pruning) |
| OS | Ubuntu 22.04 / 24.04 |

> Running a **DA node with pruning** - the recommended option for validators who also want to serve the Query API.

---

## 🚀 Quick Start

```bash
# Start the node
cd ~/espresso-decaf && docker compose up -d

# Check status
docker compose ps
curl -s http://localhost:18080/v1/status/block-height

# View live logs
docker compose logs -f sequencer
```

See [`docs/node-commands.md`](docs/node-commands.md) for the full operations cheatsheet.

---

## 📖 Documentation

| Guide | Network | Description |
|-------|---------|-------------|
| [Validator Setup - Decaf Testnet](docs/validator-setup.md) | Testnet | Step-by-step: keys, registration, DA node, and stake delegation |
| [Validator Setup - Mainnet](https://github.com/Cumulo-pro/cumulo-espresso-infra/blob/main/docs/espresso-mainnet-validator-guide.md) | Mainnet | Step-by-step: keys, registration, DA node, and stake delegation |
| [Node Commands](docs/node-commands.md) | Both | Day-to-day operations: logs, status, updates, database |
| [Metrics & Monitoring](docs/metrics-setup.md) | Both | Prometheus scraping + Grafana dashboard import |

---

## 🔗 Network Info

### Contracts (Sepolia)

| Contract | Address |
|----------|---------|
| Stake Table | `0x40304fbe94d5e7d1492dd90c53a2d63e8506a037` |
| ESP Token | `0xb3e655a030e2e34a18b72757b40be086a8f43f3b` |
| Fee Contract | `0x42835083fd1d3fc5d799b5f6815ae4bf2623e6d0` |
| Light Client (Sepolia) | `0x303872bb82a191771321d4828888920100d0b3e4` |
| Light Client (Arb Sepolia) | `0x08d16cb8243b3e172dddcdf1a1a5dacca1cd7098` |

**Decaf Chain ID:** `912559`

### Public Endpoints

| Service | URL |
|---------|-----|
| Block Explorer | https://explorer.decaf.testnet.espresso.network |
| Query API | https://query.decaf.testnet.espresso.network |
| State Relay | https://state-relay.decaf.testnet.espresso.network |

---

## 📊 Monitoring

Metrics are exposed at:

```
http://<NODE_IP>:<API_PORT>/v1/status/metrics
```

### Healthy Node - Expected Values

| Metric | Healthy Value |
|--------|--------------|
| `up` | 1 |
| `consensus_libp2p_num_connected_peers` | > 5 |
| `consensus_number_of_timeouts` | Low and stable |
| `scanner_missing_blocks` | 0 |
| `scanner_missing_vid` | 0 |
| `aggregator_height` | Increasing over time |

See [`docs/metrics-setup.md`](docs/metrics-setup.md) for full Prometheus + Grafana setup.

---

## 🔗 Resources

| Resource | URL |
|----------|-----|
| Official Docs | https://docs.espressosys.com/network |
| GitHub | https://github.com/EspressoSystems/espresso-network |
| Docker Images | https://github.com/EspressoSystems/espresso-network/pkgs/container/espresso-sequencer%2Fsequencer |
| Staking CLI README | https://github.com/EspressoSystems/espresso-network/blob/main/staking-cli/README.md |
| Discord | #decaf-node-operators |

---

*Maintained by [Cumulo](https://cumulo.pro) - trusted validator across the blockchain ecosystem.*
