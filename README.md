# Republic AI Testnet Node Installation Guide

> **Ubuntu 22.04 Compatible** - Includes GLIBC 2.38+ fix using patchelf method

This guide helps you run Republic AI testnet node on Ubuntu 22.04 by solving the GLIBC version incompatibility issue.

## 📋 Network Information

| Property | Value |
|----------|-------|
| Chain ID | `raitestnet_77701-1` |
| EVM Chain ID | `77701` |
| Denom | `arai` (base), `RAI` (display) |
| Decimals | `18` |
| Min Gas Price | `250000000arai` |
| Binary Version | `v0.1.0` |

### Public Endpoints

| Service | URL |
|---------|-----|
| Cosmos RPC | https://rpc.republicai.io |
| REST API | https://rest.republicai.io |
| gRPC | grpc.republicai.io:443 |
| EVM JSON-RPC | https://evm-rpc.republicai.io |

---

## 🔧 The GLIBC Problem & Solution

### Problem
The official `republicd` binary requires GLIBC 2.38+, but Ubuntu 22.04 only has GLIBC 2.35.

```
republicd: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

### Solution
We use **patchelf** to make the binary use a separate GLIBC 2.39 library without touching the system GLIBC. This keeps your system and other applications completely safe.

```
System GLIBC (untouched)     Separate GLIBC 2.39
/lib/x86_64-linux-gnu/       /opt/glibc-2.39/lib/
├── libc.so.6 (2.35)         ├── libc.so.6 (2.39)
│                            │
│ ← Other apps use this      │ ← Only republicd uses this
```

---

## 🚀 Installation Steps

### Step 1: Set Your Port Prefix

Choose a 2-digit port prefix (e.g., `36`, `37`, `38`) that doesn't conflict with other services.

```bash
# Set your port prefix (change XX to your preferred 2-digit number)
PORT_PREFIX=36
```

**Common port prefixes for reference:**
- Default Cosmos: `26` (26656, 26657...)
- Example custom: `36`, `37`, `38`, `39`...

### Step 2: Set Variables

```bash
# Node configuration
MONIKER="your-node-name"
CHAIN_ID="raitestnet_77701-1"
REPUBLIC_HOME="$HOME/.republicd"

# Ports (automatically calculated from prefix)
P2P_PORT="${PORT_PREFIX}656"
RPC_PORT="${PORT_PREFIX}657"
GRPC_PORT="${PORT_PREFIX}090"
API_PORT="${PORT_PREFIX}317"
PPROF_PORT="${PORT_PREFIX}060"
PROMETHEUS_PORT="${PORT_PREFIX}660"
EVM_RPC_PORT="${PORT_PREFIX}545"
EVM_WS_PORT="${PORT_PREFIX}546"

# Save to profile
echo "export REPUBLIC_HOME=$REPUBLIC_HOME" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Step 3: Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git jq patchelf
```

### Step 4: Install GLIBC 2.39 (Isolated)

```bash
# Download pre-built GLIBC 2.39
cd $HOME
wget -O glibc-2.39-ubuntu24.tar.gz https://raw.githubusercontent.com/coinsspor/coinsspor/main/glibc-2.39-ubuntu24.tar.gz

# Extract
tar -xzvf glibc-2.39-ubuntu24.tar.gz

# Move to /opt (isolated location)
sudo mkdir -p /opt/glibc-2.39/lib
sudo mv glibc-transfer/* /opt/glibc-2.39/lib/

# Verify
/opt/glibc-2.39/lib/ld-linux-x86-64.so.2 --version

# Cleanup
rm -rf glibc-transfer glibc-2.39-ubuntu24.tar.gz
```

You should see: `ld.so (Ubuntu GLIBC 2.39...) stable release version 2.39`

### Step 5: Install Go (if not installed)

```bash
# Check if Go is installed
go version || {
    cd $HOME
    GO_VERSION="1.22.5"
    wget "https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz"
    sudo rm -rf /usr/local/go
    sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
    rm "go${GO_VERSION}.linux-amd64.tar.gz"
    
    echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
    echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
    echo 'export GO111MODULE=on' >> $HOME/.bash_profile
    echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
    source $HOME/.bash_profile
}

go version
```

### Step 6: Install Cosmovisor (if not installed)

```bash
which cosmovisor || go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```

### Step 7: Download and Patch Republic Binary

```bash
# Download binary
cd $HOME
VERSION="v0.1.0"
curl -L "https://media.githubusercontent.com/media/RepublicAI/networks/main/testnet/releases/${VERSION}/republicd-linux-amd64" -o republicd
chmod +x republicd

# Patch binary to use GLIBC 2.39
patchelf --set-interpreter /opt/glibc-2.39/lib/ld-linux-x86-64.so.2 republicd
patchelf --set-rpath /opt/glibc-2.39/lib republicd

# Move to bin
sudo mv republicd /usr/local/bin/republicd

# Test - should show version without GLIBC error
republicd version
```

### Step 8: Initialize Node

```bash
# Initialize
republicd init $MONIKER --chain-id $CHAIN_ID --home $REPUBLIC_HOME

# Download genesis
curl -s https://raw.githubusercontent.com/RepublicAI/networks/main/testnet/genesis.json > $REPUBLIC_HOME/config/genesis.json

# Verify genesis
sha256sum $REPUBLIC_HOME/config/genesis.json
```

### Step 9: Setup Cosmovisor

```bash
# Create directories
mkdir -p $REPUBLIC_HOME/cosmovisor/genesis/bin
mkdir -p $REPUBLIC_HOME/cosmovisor/upgrades

# Copy binary
cp /usr/local/bin/republicd $REPUBLIC_HOME/cosmovisor/genesis/bin/

# Verify
ls -la $REPUBLIC_HOME/cosmovisor/genesis/bin/
```

### Step 10: Configure Peers

```bash
PEERS="e281dc6e4ebf5e32fb7e6c4a111c06f02a1d4d62@3.92.139.74:26656,cfb2cb90a241f7e1c076a43954f0ee6d42794d04@54.173.6.183:26656,dc254b98cebd6383ed8cf2e766557e3d240100a9@54.227.57.160:26656"
sed -i "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $REPUBLIC_HOME/config/config.toml
```

### Step 11: Configure Custom Ports

```bash
# config.toml
sed -i "s|laddr = \"tcp://0.0.0.0:26656\"|laddr = \"tcp://0.0.0.0:$P2P_PORT\"|" $REPUBLIC_HOME/config/config.toml
sed -i "s|laddr = \"tcp://127.0.0.1:26657\"|laddr = \"tcp://127.0.0.1:$RPC_PORT\"|" $REPUBLIC_HOME/config/config.toml
sed -i "s|pprof_laddr = \"localhost:6060\"|pprof_laddr = \"localhost:$PPROF_PORT\"|" $REPUBLIC_HOME/config/config.toml
sed -i "s|prometheus_listen_addr = \":26660\"|prometheus_listen_addr = \":$PROMETHEUS_PORT\"|" $REPUBLIC_HOME/config/config.toml

# app.toml
sed -i "s|address = \"tcp://localhost:1317\"|address = \"tcp://localhost:$API_PORT\"|" $REPUBLIC_HOME/config/app.toml
sed -i "s|address = \"localhost:9090\"|address = \"localhost:$GRPC_PORT\"|" $REPUBLIC_HOME/config/app.toml
sed -i "s|address = \"127.0.0.1:8545\"|address = \"127.0.0.1:$EVM_RPC_PORT\"|" $REPUBLIC_HOME/config/app.toml
sed -i "s|ws-address = \"127.0.0.1:8546\"|ws-address = \"127.0.0.1:$EVM_WS_PORT\"|" $REPUBLIC_HOME/config/app.toml

# Verify ports
echo "=== Your Port Configuration ==="
echo "P2P:        $P2P_PORT"
echo "RPC:        $RPC_PORT"
echo "gRPC:       $GRPC_PORT"
echo "API:        $API_PORT"
echo "EVM RPC:    $EVM_RPC_PORT"
echo "EVM WS:     $EVM_WS_PORT"
echo "Prometheus: $PROMETHEUS_PORT"
```

### Step 12: Configure Gas Price and Pruning

```bash
# Set minimum gas price
sed -i 's/minimum-gas-prices = "0arai"/minimum-gas-prices = "250000000arai"/' $REPUBLIC_HOME/config/app.toml

# Configure pruning (optional - saves disk space)
sed -i -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $REPUBLIC_HOME/config/app.toml
```

### Step 13: Create Systemd Service

```bash
sudo tee /etc/systemd/system/republicd.service > /dev/null << EOF
[Unit]
Description=Republic AI Node
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --home $REPUBLIC_HOME
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$REPUBLIC_HOME"
Environment="DAEMON_NAME=republicd"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF
```

### Step 14: Start Node

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service
sudo systemctl enable republicd

# Start node
sudo systemctl start republicd

# Check logs
sudo journalctl -u republicd -f -o cat
```

---

## ✅ Verify Installation

### Check Sync Status

```bash
republicd status --home $REPUBLIC_HOME 2>&1 | jq '.sync_info'
```

### Check Service Status

```bash
sudo systemctl status republicd
```

### Check Logs

```bash
sudo journalctl -u republicd -f -o cat
```

---

## 👛 Wallet Operations

### Create New Wallet

```bash
republicd keys add wallet --home $REPUBLIC_HOME
```

> ⚠️ **IMPORTANT:** Save your mnemonic phrase securely!

### Import Existing Wallet

```bash
republicd keys add wallet --recover --home $REPUBLIC_HOME
```

### Check Wallet Address

```bash
republicd keys show wallet -a --home $REPUBLIC_HOME
```

### Check Balance

```bash
republicd query bank balances $(republicd keys show wallet -a --home $REPUBLIC_HOME) --home $REPUBLIC_HOME
```

---

## 🗳️ Create Validator

> Wait until your node is fully synced (`catching_up: false`)

### Get Testnet Tokens

Join Discord and request tokens from faucet: https://discord.com/invite/therepublic

### Create Validator

```bash
republicd tx staking create-validator \
  --amount=1000000000000000000000arai \
  --pubkey=$(republicd comet show-validator --home $REPUBLIC_HOME) \
  --moniker="$MONIKER" \
  --identity="" \
  --details="Your validator description" \
  --website="https://your-website.com" \
  --chain-id=$CHAIN_ID \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --gas=auto \
  --gas-adjustment=1.5 \
  --gas-prices="250000000arai" \
  --from=wallet \
  --home $REPUBLIC_HOME \
  -y
```

> **Note:** Minimum self-delegation is 1000 RAI = `1000000000000000000000arai`

---

## 📚 Useful Commands

### Service Management

```bash
# Check status
sudo systemctl status republicd

# Restart
sudo systemctl restart republicd

# Stop
sudo systemctl stop republicd

# View logs
sudo journalctl -u republicd -f -o cat
```

### Staking Operations

```bash
# Delegate
republicd tx staking delegate <VALIDATOR_ADDRESS> <AMOUNT>arai \
  --from wallet --chain-id $CHAIN_ID \
  --gas auto --gas-adjustment 1.5 --gas-prices 250000000arai \
  --home $REPUBLIC_HOME -y

# Unjail
republicd tx slashing unjail \
  --from wallet --chain-id $CHAIN_ID \
  --gas auto --gas-adjustment 1.5 --gas-prices 250000000arai \
  --home $REPUBLIC_HOME -y

# Withdraw rewards
republicd tx distribution withdraw-all-rewards \
  --from wallet --chain-id $CHAIN_ID \
  --gas auto --gas-adjustment 1.5 --gas-prices 250000000arai \
  --home $REPUBLIC_HOME -y
```

### Node Info

```bash
# Sync status
republicd status --home $REPUBLIC_HOME 2>&1 | jq '.sync_info'

# Node ID
republicd comet show-node-id --home $REPUBLIC_HOME

# Validator info
republicd query staking validator $(republicd keys show wallet --bech val -a --home $REPUBLIC_HOME) --home $REPUBLIC_HOME
```

---

## 🔄 Upgrade Binary (Future Updates)

When a new version is released:

```bash
# Download new binary
curl -L "https://media.githubusercontent.com/media/RepublicAI/networks/main/testnet/releases/NEW_VERSION/republicd-linux-amd64" -o republicd_new
chmod +x republicd_new

# Patch it
patchelf --set-interpreter /opt/glibc-2.39/lib/ld-linux-x86-64.so.2 republicd_new
patchelf --set-rpath /opt/glibc-2.39/lib republicd_new

# For Cosmovisor auto-upgrade, place in upgrades folder:
mkdir -p $REPUBLIC_HOME/cosmovisor/upgrades/<upgrade-name>/bin
mv republicd_new $REPUBLIC_HOME/cosmovisor/upgrades/<upgrade-name>/bin/republicd
```

---

## 🛠️ Troubleshooting

### GLIBC Error Still Appears

```bash
# Verify patchelf settings
patchelf --print-interpreter /usr/local/bin/republicd
patchelf --print-rpath /usr/local/bin/republicd

# Should show:
# /opt/glibc-2.39/lib/ld-linux-x86-64.so.2
# /opt/glibc-2.39/lib
```

### Port Conflicts

```bash
# Check if ports are in use
sudo lsof -i :${P2P_PORT}
sudo lsof -i :${RPC_PORT}
```

### Node Not Syncing

```bash
# Check peers
curl -s localhost:${RPC_PORT}/net_info | jq '.result.n_peers'

# Add more peers manually if needed
```

---

## 🔗 Resources

- **GitHub:** https://github.com/RepublicAI/networks
- **Discord:** https://discord.com/invite/therepublic
- **Documentation:** https://github.com/RepublicAI/networks/issues

---

## 📝 Credits

- GLIBC 2.39 fix method by [@coinsspor](https://github.com/coinsspor)
- Republic AI Team for the testnet

---

*Last updated: January 2026*
