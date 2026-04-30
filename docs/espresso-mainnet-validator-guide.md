# Espresso Network — Mainnet 1 Validator Setup Guide

> **By Cumulo** | April 2026  
> A step-by-step guide to register and run a DA node validator on Espresso Mainnet 1.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Hardware Requirements](#hardware-requirements)
4. [Install Docker](#install-docker)
5. [Generate Validator Keys](#generate-validator-keys)
6. [Create Your Validator Wallet](#create-your-validator-wallet)
7. [Fund the Validator Wallet](#fund-the-validator-wallet)
8. [Create Validator Metadata](#create-validator-metadata)
9. [Register as Validator On-Chain](#register-as-validator-on-chain)
10. [Set Up the DA Node](#set-up-the-da-node)
11. [Start the Node](#start-the-node)
12. [Self-Delegate ESP Tokens](#self-delegate-esp-tokens)
13. [Verify Your Node](#verify-your-node)
14. [Useful Commands](#useful-commands)
15. [Network Contracts](#network-contracts)
16. [Resources](#resources)

---

## Overview

**Espresso Network Mainnet 1** operates a permissionless Proof-of-Stake consensus using the HotShot protocol. The active validator set consists of the **top 100 nodes by delegated ESP stake**, updated dynamically each epoch (~24 hours).

Unlike Decaf testnet, Mainnet requires **real ESP tokens** for delegation — either self-delegated or from external delegators. There is no team-managed delegation for Mainnet.

### Key Differences from Decaf Testnet

| | Decaf (Testnet) | Mainnet 1 |
|--|-----------------|-----------|
| L1 | Sepolia | **Ethereum Mainnet** |
| Image tag | `20260424` | **`20260407`** |
| Genesis file | `decaf.toml` | **`mainnet.toml`** |
| Stake | Team-delegated (free) | **Real ESP tokens required** |
| Entry to active set | Request to team | **Top 100 by ESP stake** |
| Epoch duration | ~24h | **~24h** |

---

## Prerequisites

- A Linux server (Ubuntu 22.04 or 24.04 recommended)
- Docker installed
- A dedicated Ethereum wallet with ETH Mainnet for gas
- An Ethereum Mainnet L1 RPC endpoint (see [L1 Provider](#l1-provider) section)
- ESP tokens for delegation (ERC-20 on Ethereum Mainnet)
- Port **1769/tcp** open for libp2p P2P
- An available port for the Query API (e.g. 18080)

### L1 Provider

Espresso requires `eth_getLogs` with wide block ranges. Recommended providers:

| Provider | Cost | Notes |
|----------|------|-------|
| **LlamaRPC** `https://eth.llamarpc.com` | Free | No API key needed |
| **Alchemy PAYG** | ~cents/month | Most reliable |
| **Infura** | Free tier | May have range limits |

> ⚠️ Avoid Alchemy's free tier — it limits `eth_getLogs` to 10-block ranges which is insufficient for Espresso.

---

## Hardware Requirements

| Component | Sequencer | PostgreSQL | Total |
|-----------|-----------|------------|-------|
| CPU | 4 cores | 2 cores | **6 cores** |
| RAM | 8 GB | 4 GB | **12 GB** |
| Disk | 100 GB SSD | (included) | **100 GB** |

---

## Install Docker

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

Verify:

```bash
docker --version && docker compose version
```

> **Troubleshooting:** If Docker fails with iptables errors on older kernels:
> ```bash
> sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
> sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
> sudo systemctl restart docker
> ```
> If the issue persists, a kernel update and reboot are required.

---

## Generate Validator Keys

Create the working directory structure:

```bash
mkdir -p ~/espresso-mainnet/keys
mkdir -p ~/espresso-mainnet/data/postgres
```

Generate validator keys — BLS consensus key and Schnorr state key:

```bash
docker run --rm \
  -v ~/espresso-mainnet/keys:/output \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260407 \
  keygen --out /output
```

View the generated keys:

```bash
cat ~/espresso-mainnet/keys/0.env
```

Output:

```
# Seed: <random seed>
ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY=BLS_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY=BLS_SIGNING_KEY~...
ESPRESSO_SEQUENCER_PUBLIC_STATE_KEY=SCHNORR_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY=SCHNORR_SIGNING_KEY~...
```

> ⚠️ **Back up your seed and all private keys immediately. They cannot be recovered if lost.**

> **Note:** Mainnet uses a `Seed` instead of a `Mnemonic` for key generation. The validator wallet is created separately.

---

## Create Your Validator Wallet

Generate a dedicated Ethereum wallet for signing the on-chain registration:

```bash
pip3 install eth-account --break-system-packages

python3 -c "
from eth_account import Account
import secrets
Account.enable_unaudited_hdwallet_features()
mnemonic = Account.create_with_mnemonic()[1]
acct = Account.from_mnemonic(mnemonic, account_path=\"m/44'/60'/0'/0/0\")
print('Mnemonic:', mnemonic)
print('Address:', acct.address)
"
```

> ⚠️ **Save the mnemonic securely. This wallet will also receive your validator commission.**

---

## Fund the Validator Wallet

Send ETH to your validator wallet address to cover gas for:
- Registration transaction (~$2-4)
- Approve transaction (~$1-2)
- Delegate transaction (~$2-4)

**Recommended: send at least 0.01 ETH.**

Verify the balance:

```bash
python3 -c "
import urllib.request, json
url = 'https://<YOUR_RPC_ENDPOINT>'
data = json.dumps({'id':1,'jsonrpc':'2.0','method':'eth_getBalance','params':['<YOUR_VALIDATOR_ADDRESS>','latest']}).encode()
req = urllib.request.Request(url, data=data, headers={'Content-Type':'application/json'})
res = json.loads(urllib.request.urlopen(req).read())
balance = int(res['result'], 16) / 1e18
print(f'Balance: {balance:.6f} ETH')
"
```

---

## Create Validator Metadata

Create a GitHub Gist at https://gist.github.com with the filename `espresso-mainnet-metadata.json`.

The Espresso team reviews validator metadata for their delegation dashboard. Include icons for best results:

```json
{
  "name": "Your Validator Name",
  "description": "A brief description of your validator.",
  "pub_key": "BLS_VER_KEY~<YOUR_ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY>",
  "icons": {
    "small": {
      "1x": "https://your-domain.com/icon-14x14.png",
      "2x": "https://your-domain.com/icon-28x28.png",
      "3x": "https://your-domain.com/icon-42x42.png"
    },
    "medium": {
      "1x": "https://your-domain.com/icon-24x24.png",
      "2x": "https://your-domain.com/icon-48x48.png",
      "3x": "https://your-domain.com/icon-72x72.png"
    }
  }
}
```

- Set the gist to **Public**
- Copy the **Raw** URL after saving

Verify it is accessible:

```bash
curl -s "<YOUR_RAW_GIST_URL>"
```

> **Icon sizes required:** 14x14, 28x28, 42x42 (small), 24x24, 48x48, 72x72 (medium). All PNG with transparent background.

---

## Register as Validator On-Chain

Load your keys and run the registration. This is a **one-time on-chain transaction** on Ethereum Mainnet:

```bash
source ~/espresso-mainnet/keys/0.env

docker run --rm \
  -e L1_PROVIDER="<YOUR_ETHEREUM_MAINNET_RPC>" \
  -e STAKE_TABLE_ADDRESS=0xCeF474D372B5b09dEfe2aF187bf17338Dc704451 \
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
event: ValidatorRegisteredV2 { account: 0x..., blsVK: BLS_VER_KEY~..., commission: 1000, metadataUri: ... }
```

Verify on Etherscan:
`https://etherscan.io/tx/<YOUR_TX_HASH>`

---

## Set Up the DA Node

Create the `docker-compose.yml`:

```bash
cat > ~/espresso-mainnet/docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_USER: espresso
      POSTGRES_PASSWORD: <CHOOSE_A_STRONG_PASSWORD>
      POSTGRES_DB: espresso_mainnet
    volumes:
      - /home/<YOUR_USER>/espresso-mainnet/data/postgres:/var/lib/postgresql/data

  sequencer:
    image: ghcr.io/espressosystems/espresso-sequencer/sequencer:20260407
    restart: unless-stopped
    depends_on:
      - postgres
    command: sequencer -- storage-sql -- http -- catchup -- status -- query
    ports:
      - "1769:1769/tcp"
      - "18080:80"
    volumes:
      - /home/<YOUR_USER>/espresso-mainnet/keys:/keys:ro
    environment:
      ESPRESSO_SEQUENCER_CDN_ENDPOINT: "cdn.main.net.espresso.network:1737"
      ESPRESSO_STATE_RELAY_SERVER_URL: "https://state-relay.main.net.espresso.network"
      ESPRESSO_SEQUENCER_GENESIS_FILE: "/genesis/mainnet.toml"
      ESPRESSO_SEQUENCER_CONFIG_PEERS: "https://cache.main.net.espresso.network"
      ESPRESSO_SEQUENCER_KEY_FILE: "/keys/0.env"
      ESPRESSO_SEQUENCER_STATE_PEERS: "https://query.main.net.espresso.network/v1"
      ESPRESSO_SEQUENCER_API_PEERS: "https://query.main.net.espresso.network/v1"
      ESPRESSO_SEQUENCER_POSTGRES_PRUNE: "true"
      ESPRESSO_SEQUENCER_IS_DA: "true"
      ESPRESSO_SEQUENCER_L1_PROVIDER: "<YOUR_ETHEREUM_MAINNET_HTTP_RPC>"
      ESPRESSO_SEQUENCER_L1_WS_PROVIDER: "<YOUR_ETHEREUM_MAINNET_WSS_RPC>"
      ESPRESSO_SEQUENCER_L1_RETRY_DELAY: "20s"
      ESPRESSO_SEQUENCER_POSTGRES_HOST: "postgres"
      ESPRESSO_SEQUENCER_POSTGRES_USER: "espresso"
      ESPRESSO_SEQUENCER_POSTGRES_PASSWORD: "<CHOOSE_A_STRONG_PASSWORD>"
      ESPRESSO_SEQUENCER_POSTGRES_DATABASE: "espresso_mainnet"
      ESPRESSO_SEQUENCER_LIBP2P_BIND_ADDRESS: "0.0.0.0:1769"
      ESPRESSO_SEQUENCER_LIBP2P_ADVERTISE_ADDRESS: "<YOUR_PUBLIC_IP>:1769"
      ESPRESSO_SEQUENCER_API_PORT: "80"
      RUST_LOG: "warn,libp2p=off"
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
| `<CHOOSE_A_STRONG_PASSWORD>` | A secure PostgreSQL password (same in both places) |
| `<YOUR_USER>` | Your Linux username |
| `<YOUR_PUBLIC_IP>` | Your server's public IP address |
| `<YOUR_ETHEREUM_MAINNET_HTTP_RPC>` | e.g. `https://eth.llamarpc.com` |
| `<YOUR_ETHEREUM_MAINNET_WSS_RPC>` | e.g. `wss://eth.llamarpc.com` |

> **Important:** Mainnet DA nodes require `ESPRESSO_SEQUENCER_IS_DA: "true"` — this is not needed in Decaf.

> **API peers:** Note that Mainnet uses `/v1` suffix in state and API peers URLs, unlike Decaf.

---

## Start the Node

```bash
cd ~/espresso-mainnet && docker compose up -d
```

Verify both containers are running:

```bash
docker compose ps
```

---

## Self-Delegate ESP Tokens

To enter the active validator set you need ESP tokens delegated to your validator. You can self-delegate if you hold ESP tokens.

ESP is an ERC-20 token on **Ethereum Mainnet**:
- Contract: `0x031De51F3E8016514Bd0963d0B2AB825A591Db9A`
- Available on: Binance, Coinbase, Kraken, KuCoin, Uniswap

Send ESP to your validator wallet address, then:

**Step 1 — Approve the stake table contract to spend your ESP:**

```bash
docker run --rm \
  -e MNEMONIC="<YOUR_12_WORD_MNEMONIC>" \
  -e ACCOUNT_INDEX=0 \
  -e L1_PROVIDER="<YOUR_ETHEREUM_MAINNET_RPC>" \
  -e ESP_TOKEN_ADDRESS=0x031De51F3E8016514Bd0963d0B2AB825A591Db9A \
  -e STAKE_TABLE_ADDRESS=0xCeF474D372B5b09dEfe2aF187bf17338Dc704451 \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli approve \
  --amount <AMOUNT>
```

**Step 2 — Delegate to your validator:**

```bash
docker run --rm \
  -e MNEMONIC="<YOUR_12_WORD_MNEMONIC>" \
  -e ACCOUNT_INDEX=0 \
  -e L1_PROVIDER="<YOUR_ETHEREUM_MAINNET_RPC>" \
  -e ESP_TOKEN_ADDRESS=0x031De51F3E8016514Bd0963d0B2AB825A591Db9A \
  -e STAKE_TABLE_ADDRESS=0xCeF474D372B5b09dEfe2aF187bf17338Dc704451 \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli delegate \
  --validator-address <YOUR_VALIDATOR_ADDRESS> \
  --amount <AMOUNT>
```

Once delegated, your node will enter the active set at the next epoch change (~24 hours).

> **Note:** The approve and delegate steps each require ETH for gas on Ethereum Mainnet (~$3-6 total depending on gas price).

---

## Verify Your Node

Check block height (increases once in the active set):

```bash
curl -s http://localhost:18080/v1/status/block-height
```

Check key metrics:

```bash
curl -s http://localhost:18080/v1/status/metrics | grep -E "^aggregator_height|^consensus_libp2p_num_connected_peers|^consensus_current_view"
```

View your validator on the Mainnet explorer:
https://explorer.main.net.espresso.network

> **Expected behavior before entering active set:** Block height stays at 1, no peers connected. This is normal — the node joins consensus at the next epoch after delegation.

> **Expected warning after registration (normal):** `LCV3 signature posted by nodes not on the stake table` — this disappears once the new stake table takes effect at the next epoch.

---

## Useful Commands

```bash
# Start the node
cd ~/espresso-mainnet && docker compose up -d

# Stop the node
docker compose down

# View logs
docker compose logs -f sequencer

# View only errors
docker compose logs -f sequencer 2>&1 | grep ERROR

# Check block height
curl -s http://localhost:18080/v1/status/block-height

# Update to a new image version
sed -i 's/sequencer:20260407/sequencer:<NEW_TAG>/g' ~/espresso-mainnet/docker-compose.yml
docker compose pull && docker compose up -d

# Update commission rate
docker run --rm \
  -e MNEMONIC="<YOUR_MNEMONIC>" \
  -e ACCOUNT_INDEX=0 \
  -e L1_PROVIDER="<YOUR_RPC>" \
  -e STAKE_TABLE_ADDRESS=0xCeF474D372B5b09dEfe2aF187bf17338Dc704451 \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli update-commission \
  --new-commission <NEW_RATE>

# Deregister validator
docker run --rm \
  -e MNEMONIC="<YOUR_MNEMONIC>" \
  -e ACCOUNT_INDEX=0 \
  -e L1_PROVIDER="<YOUR_RPC>" \
  -e ESP_TOKEN_ADDRESS=0x031De51F3E8016514Bd0963d0B2AB825A591Db9A \
  -e STAKE_TABLE_ADDRESS=0xCeF474D372B5b09dEfe2aF187bf17338Dc704451 \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli deregister-validator
```

> **Commission update rules:** Maximum once per week. Increases capped at 5% per update. Decreases have no limit.

---

## Files to Back Up

| File | Content |
|------|---------|
| `~/espresso-mainnet/keys/0.env` | BLS and Schnorr private keys + seed |
| Validator wallet mnemonic | 12-word phrase for your Ethereum wallet |

> ⚠️ **Without these you lose control of your validator and delegated ESP. There is no recovery.**

---

## Network Contracts (Ethereum Mainnet)

| Contract | Address |
|----------|---------|
| Stake Table | `0xCeF474D372B5b09dEfe2aF187bf17338Dc704451` |
| ESP Token | `0x031De51F3E8016514Bd0963d0B2AB825A591Db9A` |

---

## Public Endpoints

| Service | URL |
|---------|-----|
| Block Explorer | https://explorer.main.net.espresso.network |
| Query API | https://query.main.net.espresso.network |
| State Relay | https://state-relay.main.net.espresso.network |
| CDN | `cdn.main.net.espresso.network:1737` |

---

## Resources

| Resource | URL |
|----------|-----|
| Official Docs | https://docs.espressosys.com/network |
| GitHub | https://github.com/EspressoSystems/espresso-network |
| Docker Images | https://github.com/EspressoSystems/espresso-network/pkgs/container/espresso-sequencer%2Fsequencer |
| Staking CLI README | https://github.com/EspressoSystems/espresso-network/blob/main/staking-cli/README.md |
| Docker Compose Examples | https://github.com/EspressoSystems/espresso-for-dummies |
| X (Twitter) | https://x.com/EspressoSys |
| Discord | #mainnet-node-operators |

---

*Guide maintained by [Cumulo](https://cumulo.pro) — trusted validator across the blockchain ecosystem.*
