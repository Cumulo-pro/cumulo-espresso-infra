# Espresso Network — Decaf Testnet Validator Setup Guide

> **By Cumulo** | April 2026
> A step-by-step guide to register and run a DA node validator on Espresso's **Decaf testnet**.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Hardware Requirements](#hardware-requirements)
4. [Install Docker](#install-docker)
5. [Generate Validator Keys](#generate-validator-keys)
6. [Derive Your Validator Wallet Address](#derive-your-validator-wallet-address)
7. [Fund the Validator Wallet](#fund-the-validator-wallet)
8. [Create Validator Metadata](#create-validator-metadata)
9. [Register as Validator On-Chain](#register-as-validator-on-chain)
10. [Set Up the DA Node](#set-up-the-da-node)
11. [Start the Node](#start-the-node)
12. [Verify Your Node](#verify-your-node)
13. [Request Stake Delegation](#request-stake-delegation)
14. [Useful Commands](#useful-commands)
15. [Network Contracts](#network-contracts)
16. [Resources](#resources)

---

## Overview

**Espresso Network** is a decentralized confirmation layer for Ethereum rollups. It uses the HotShot consensus protocol to confirm blocks in a fast, trust-minimized way without relying on a centralized sequencer.

**Decaf** is Espresso's persistent long-running testnet, launched in September 2024 and upgraded to Proof-of-Stake (PoS) with HotShot in April 2025. It mirrors the Mainnet 1.0 model where the top 100 validators by stake participate in consensus.

### Node Types

| Type | Consensus | Query API | Disk | PostgreSQL |
|------|-----------|-----------|------|------------|
| Regular node | ✅ | ❌ | ~MB | ❌ |
| DA node (pruning) | ✅ | ✅ | 100 GB | ✅ |
| DA node (archival) | ✅ | ✅ | 1.2 TB+ | ✅ |

This guide covers setting up a **DA node with pruning** — the recommended option for validators who also want to serve the Query API.

> **Note on access:** Decaf is a semi-permissioned testnet. The Espresso team manually delegates testnet stake to new operators. You can register on-chain without prior permission, but you will need stake delegated by the team to join the active consensus set.

---

## Prerequisites

- A Linux server (Ubuntu 22.04 or 24.04 recommended)
- Docker installed
- A dedicated Ethereum wallet with Sepolia ETH for gas (do not reuse your main wallet)
- Access to a Sepolia L1 RPC endpoint (PublicNode recommended — free, no rate limits)
- Port **1769/tcp** open for libp2p P2P
- An available port for the Query API (default 8080, use an alternative if already occupied)

---

## Hardware Requirements

| Component | Sequencer | PostgreSQL | Total |
|-----------|-----------|------------|-------|
| CPU | 4 cores | 2 cores | **6 cores** |
| RAM | 8 GB | 4 GB | **12 GB** |
| Disk | 100 GB SSD | (included) | **100 GB** |

---

## Install Docker

If Docker is not already installed, add the official Docker repository:

```bash
# Remove conflicting packages
sudo apt-get remove -y docker.io docker-doc docker-compose podman-docker containerd runc 2>/dev/null
sudo apt-get remove -y docker-compose-v2 2>/dev/null

# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start Docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

Verify installation:

```bash
docker --version && docker compose version
```

> **Troubleshooting — iptables error on older kernels:**
> If Docker fails to start with iptables/nftables errors, switch to legacy mode:
> ```bash
> sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
> sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
> sudo systemctl restart docker
> ```
> If the error persists (`bridge docker0 operation not supported`), a kernel update and reboot are required. Before rebooting, ensure all services on the server are managed by systemd with `Restart=on-failure` and `WantedBy=multi-user.target` so they recover automatically.

---

## Generate Validator Keys

Create the working directory structure:

```bash
mkdir -p ~/espresso-decaf/keys
mkdir -p ~/espresso-decaf/data/postgres
```

Generate the three required keys — Ethereum wallet mnemonic, BLS consensus key, and Schnorr state key:

```bash
docker run --rm \
  -v ~/espresso-decaf/keys:/output \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260424 \
  keygen --out /output
```

View the generated keys:

```bash
cat ~/espresso-decaf/keys/0.env
```

The output will look like this:

```
# Mnemonic: word1 word2 word3 ... word12
# Index: 0
ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY=BLS_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY=BLS_SIGNING_KEY~...
ESPRESSO_SEQUENCER_PUBLIC_STATE_KEY=SCHNORR_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY=SCHNORR_SIGNING_KEY~...
ESPRESSO_SEQUENCER_PUBLIC_X25519_KEY=...
ESPRESSO_SEQUENCER_PRIVATE_X25519_KEY=X25519_SK~...
```

> ⚠️ **Back up your mnemonic and all private keys immediately and securely. They cannot be recovered if lost.**

---

## Derive Your Validator Wallet Address

The mnemonic generates a deterministic Ethereum address that signs the on-chain registration transaction. Derive it with Python:

```bash
pip3 install eth-account --break-system-packages

python3 -c "
from eth_account import Account
Account.enable_unaudited_hdwallet_features()
acct = Account.from_mnemonic('<YOUR_MNEMONIC_HERE>', account_path=\"m/44'/60'/0'/0/0\")
print('Address:', acct.address)
"
```

Save this address — this is your **validator wallet address** on Sepolia.

---

## Fund the Validator Wallet

Send ~0.05 ETH Sepolia to your validator wallet address to cover gas for the registration transaction.

Sepolia faucets:
- https://sepoliafaucet.com
- https://faucet.quicknode.com/ethereum/sepolia
- https://faucets.chain.link/sepolia

Verify the balance:

```bash
python3 -c "
import urllib.request, json
url = 'https://ethereum-sepolia-rpc.publicnode.com'
data = json.dumps({'id':1,'jsonrpc':'2.0','method':'eth_getBalance','params':['<YOUR_VALIDATOR_ADDRESS>','latest']}).encode()
req = urllib.request.Request(url, data=data, headers={'Content-Type':'application/json'})
res = json.loads(urllib.request.urlopen(req).read())
balance = int(res['result'], 16) / 1e18
print(f'Balance: {balance:.6f} ETH Sepolia')
"
```

---

## Create Validator Metadata

The registration requires a publicly accessible JSON metadata file. Create a GitHub Gist at https://gist.github.com with the filename `espresso-metadata.json` and the following content:

```json
{
  "name": "Your Validator Name",
  "description": "A brief description of your validator.",
  "pub_key": "BLS_VER_KEY~<YOUR_ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY>"
}
```

- Set the gist visibility to **Public**
- After saving, click the **Raw** button and copy the URL

Verify it is accessible:

```bash
curl -s "<YOUR_RAW_GIST_URL>"
```

> **Important:** The `pub_key` field must contain your full `ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY` value including the `BLS_VER_KEY~` prefix. Without it the registration will fail with a schema validation error.

---

## Register as Validator On-Chain

Load your keys and run the registration. This is a **one-time on-chain transaction** on Sepolia:

```bash
source ~/espresso-decaf/keys/0.env

docker run --rm \
  -e L1_PROVIDER="https://ethereum-sepolia-rpc.publicnode.com" \
  -e STAKE_TABLE_ADDRESS=0x40304fbe94d5e7d1492dd90c53a2d63e8506a037 \
  -e MNEMONIC="<YOUR_12_WORD_MNEMONIC>" \
  -e ACCOUNT_INDEX=0 \
  -e CONSENSUS_PRIVATE_KEY="$ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY" \
  -e STATE_PRIVATE_KEY="$ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY" \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli register-validator \
  --commission 10 \
  --metadata-uri "<YOUR_RAW_GIST_URL>"
```

A successful registration returns:

```
Success! transaction hash: 0x...
event: ValidatorRegisteredV2 { account: 0x..., blsVK: BLS_VER_KEY~..., schnorrVK: SCHNORR_VER_KEY~..., commission: 1000, metadataUri: ... }
```

> `--commission 10` sets a 10% commission (stored as 1000 basis points). Adjust as needed.

Verify on Sepolia Etherscan:
`https://sepolia.etherscan.io/tx/<YOUR_TX_HASH>`

---

## Set Up the DA Node

Create the `docker-compose.yml`:

```bash
cat > ~/espresso-decaf/docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_USER: espresso
      POSTGRES_PASSWORD: <CHOOSE_A_STRONG_PASSWORD>
      POSTGRES_DB: espresso_decaf
    volumes:
      - /home/<YOUR_USER>/espresso-decaf/data/postgres:/var/lib/postgresql/data

  sequencer:
    image: ghcr.io/espressosystems/espresso-sequencer/sequencer:20260424
    restart: unless-stopped
    depends_on:
      - postgres
    command: sequencer -- storage-sql -- http -- catchup -- status -- query
    ports:
      - "1769:1769/tcp"
      - "18080:80"
    volumes:
      - /home/<YOUR_USER>/espresso-decaf/keys:/keys:ro
    environment:
      ESPRESSO_SEQUENCER_ORCHESTRATOR_URL: "https://orchestrator-UZAFTUIMZOT.decaf.testnet.espresso.network/"
      ESPRESSO_SEQUENCER_CDN_ENDPOINT: "cdn.decaf.testnet.espresso.network:1737"
      ESPRESSO_STATE_RELAY_SERVER_URL: "https://state-relay.decaf.testnet.espresso.network"
      ESPRESSO_SEQUENCER_GENESIS_FILE: "/genesis/decaf.toml"
      ESPRESSO_SEQUENCER_CONFIG_PEERS: "https://cache.decaf.testnet.espresso.network"
      ESPRESSO_SEQUENCER_KEY_FILE: "/keys/0.env"
      ESPRESSO_SEQUENCER_STATE_PEERS: "https://query.decaf.testnet.espresso.network"
      ESPRESSO_SEQUENCER_API_PEERS: "https://query.decaf.testnet.espresso.network"
      ESPRESSO_SEQUENCER_POSTGRES_PRUNE: "true"
      ESPRESSO_SEQUENCER_L1_PROVIDER: "https://ethereum-sepolia-rpc.publicnode.com"
      ESPRESSO_SEQUENCER_L1_WS_PROVIDER: "wss://ethereum-sepolia-rpc.publicnode.com"
      ESPRESSO_SEQUENCER_L1_RETRY_DELAY: "20s"
      ESPRESSO_SEQUENCER_POSTGRES_HOST: "postgres"
      ESPRESSO_SEQUENCER_POSTGRES_USER: "espresso"
      ESPRESSO_SEQUENCER_POSTGRES_PASSWORD: "<CHOOSE_A_STRONG_PASSWORD>"
      ESPRESSO_SEQUENCER_POSTGRES_DATABASE: "espresso_decaf"
      ESPRESSO_SEQUENCER_LIBP2P_BIND_ADDRESS: "0.0.0.0:1769"
      ESPRESSO_SEQUENCER_LIBP2P_ADVERTISE_ADDRESS: "<YOUR_PUBLIC_IP>:1769"
      ESPRESSO_SEQUENCER_API_PORT: "80"
      RUST_LOG: "warn,hotshot_libp2p_networking=off"
      RUST_LOG_FORMAT: "json"
    sysctls:
      net.ipv4.tcp_congestion_control: "bbr"
      net.ipv4.tcp_rmem: "8192 262144 67108864"
      net.ipv4.tcp_wmem: "4096 16384 536870912"
      net.ipv4.tcp_adv_win_scale: "0"
      net.ipv4.tcp_notsent_lowat: "131072"
      net.ipv4.tcp_slow_start_after_idle: "0"
EOF
```

Replace all placeholders before starting:

| Placeholder | Replace with |
|-------------|-------------|
| `<CHOOSE_A_STRONG_PASSWORD>` | A secure PostgreSQL password (same value in both places) |
| `<YOUR_USER>` | Your Linux username |
| `<YOUR_PUBLIC_IP>` | Your server's public IP address |

> **Port note:** The example maps the Query API to port `18080` to avoid conflicts with other services. Change `18080:80` to `8080:80` if port 8080 is free on your server.

> **L1 Provider note:** Avoid Alchemy's free tier — it limits `eth_getLogs` to 10-block ranges, which is insufficient for Espresso. Use PublicNode (free, no limits) or an Alchemy PAYG plan.

---

## Start the Node

```bash
cd ~/espresso-decaf && docker compose up -d
```

Verify both containers are running:

```bash
docker compose ps
```

---

## Verify Your Node

Check the Query API:

```bash
curl -s http://localhost:18080/v1/status/block-height
```

Expected output before stake delegation: `1`

Once the team delegates stake and the node joins the active set, this value will start increasing.

View logs:

```bash
docker compose logs -f sequencer
```

**Normal warnings while waiting for stake delegation:**

| Warning | Meaning |
|---------|---------|
| `Stake table for Epoch XXXX Unavailable` | Expected — no stake delegated yet |
| `NoPeersYet` | Transient — disappears once connected to network |
| `view timed out` | Expected — not yet in consensus set |
| `Catch up already in Progress` | Normal during initial sync |

View your validator on the Decaf block explorer:
https://explorer.decaf.testnet.espresso.network

---

## Request Stake Delegation

Join the Espresso Discord and post in `#decaf-node-operators`:

```
Hi team! We just registered our validator on Decaf and our DA node with pruning is up and running.

Registration TX: <YOUR_TX_HASH>
Validator account: <YOUR_VALIDATOR_ADDRESS>
BLS key: <YOUR_BLS_VER_KEY>

We're running on dedicated hardware and ready to join the active set.
Could you delegate testnet stake to our validator? Thank you!
```

Once the team delegates stake, the block height will start increasing and your validator will appear in the explorer.

---

## Useful Commands

```bash
# Start the node
cd ~/espresso-decaf && docker compose up -d

# Stop the node
docker compose down

# View logs
docker compose logs -f sequencer

# View only errors
docker compose logs -f sequencer 2>&1 | grep ERROR

# Check block height
curl -s http://localhost:18080/v1/status/block-height

# Check genesis block
curl -s http://localhost:18080/v1/availability/block/0 | jq .

# Update to a new image version
sed -i 's/sequencer:20260424/sequencer:<NEW_TAG>/g' ~/espresso-decaf/docker-compose.yml
docker compose pull && docker compose up -d
```

---

## Network Contracts (Sepolia)

| Contract | Address |
|----------|---------|
| Stake Table | `0x40304fbe94d5e7d1492dd90c53a2d63e8506a037` |
| ESP Token | `0xb3e655a030e2e34a18b72757b40be086a8f43f3b` |
| Fee Contract | `0x42835083fd1d3fc5d799b5f6815ae4bf2623e6d0` |
| Light Client (Sepolia) | `0x303872bb82a191771321d4828888920100d0b3e4` |
| Light Client (Arb Sepolia) | `0x08d16cb8243b3e172dddcdf1a1a5dacca1cd7098` |

**Decaf Chain ID:** `912559`

---

## Public Endpoints

| Service | URL |
|---------|-----|
| Block Explorer | https://explorer.decaf.testnet.espresso.network |
| Query API | https://query.decaf.testnet.espresso.network |
| State Relay | https://state-relay.decaf.testnet.espresso.network |
| CDN | `cdn.decaf.testnet.espresso.network:1737` |

---

## Resources

| Resource | URL |
|----------|-----|
| Official Docs | https://docs.espressosys.com/network |
| GitHub | https://github.com/EspressoSystems/espresso-network |
| Docker Images | https://github.com/EspressoSystems/espresso-network/pkgs/container/espresso-sequencer%2Fsequencer |
| Staking CLI README | https://github.com/EspressoSystems/espresso-network/blob/main/staking-cli/README.md |
| X (Twitter) | https://x.com/EspressoSys |
| Discord | #decaf-node-operators |

---

*Guide maintained by [Cumulo](https://cumulo.pro) — trusted validator across the blockchain ecosystem.*
