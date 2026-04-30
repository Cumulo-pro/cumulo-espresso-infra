# Espresso Network — Node Commands Cheatsheet

> **By Cumulo** | April 2026  
> Quick reference for day-to-day node operations.

---

## 🚀 Node Lifecycle

```bash
# Start the node
cd ~/espresso-decaf && docker compose up -d

# Stop the node
cd ~/espresso-decaf && docker compose down

# Restart the node (graceful)
cd ~/espresso-decaf && docker compose restart sequencer

# Full restart (stop + start)
cd ~/espresso-decaf && docker compose down && docker compose up -d

# Check running containers
docker compose ps
```

---

## 📋 Logs

```bash
# Live logs (sequencer)
docker compose logs -f sequencer

# Live logs (postgres)
docker compose logs -f postgres

# Last 100 lines
docker compose logs --tail 100 sequencer

# Only errors
docker compose logs -f sequencer 2>&1 | grep ERROR

# Only warnings and errors
docker compose logs -f sequencer 2>&1 | grep -E "ERROR|WARN"

# Exclude noisy lines
docker compose logs -f sequencer 2>&1 | grep -v -E "NoPeersYet|Vote sending timed out"
```

---

## 📊 Node Status

```bash
# Block height
curl -s http://localhost:18080/v1/status/block-height

# Genesis block
curl -s http://localhost:18080/v1/availability/block/0 | jq .

# Latest block
curl -s http://localhost:18080/v1/availability/block/$(curl -s http://localhost:18080/v1/status/block-height) | jq .

# All Prometheus metrics
curl -s http://localhost:18080/v1/status/metrics

# Key metrics (peers, height, view)
curl -s http://localhost:18080/v1/status/metrics | grep -E "^aggregator_height|^consensus_libp2p_num_connected_peers|^consensus_current_view|^consensus_last_decided_view|^consensus_number_of_timeouts "
```

---

## 🔄 Updates

```bash
# Update to a new image version
cd ~/espresso-decaf
sed -i 's/sequencer:CURRENT_TAG/sequencer:NEW_TAG/g' docker-compose.yml
docker compose pull
docker compose up -d
```

---

## 🗄️ Database

```bash
# Connect to PostgreSQL
docker compose exec postgres psql -U espresso -d espresso_decaf

# Check database size
docker compose exec postgres psql -U espresso -d espresso_decaf -c "\l+"

# Check largest tables
docker compose exec postgres psql -U espresso -d espresso_decaf \
  -c "SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) FROM pg_catalog.pg_statsinfo_relations ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;"
```

---

## 🔑 Keys

```bash
# View public keys (safe to share)
grep "PUBLIC" ~/espresso-decaf/keys/0.env

# View mnemonic (handle with care)
grep "Mnemonic" ~/espresso-decaf/keys/0.env
```

---

## 🌐 Network

```bash
# Check port 1769 (libp2p) is listening
ss -tlnp | grep 1769

# Check port 18080 (Query API) is listening
ss -tlnp | grep 18080

# Test L1 Sepolia RPC
curl -s -X POST https://ethereum-sepolia-rpc.publicnode.com \
  -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"eth_blockNumber","params":[]}'
```

---

## 📁 Useful Paths

| Path | Description |
|------|-------------|
| `~/espresso-decaf/docker-compose.yml` | Main configuration file |
| `~/espresso-decaf/keys/0.env` | Validator keys |
| `~/espresso-decaf/data/postgres/` | PostgreSQL data directory |

---

*Cheatsheet maintained by [Cumulo](https://cumulo.pro) — trusted validator across the blockchain ecosystem.*
