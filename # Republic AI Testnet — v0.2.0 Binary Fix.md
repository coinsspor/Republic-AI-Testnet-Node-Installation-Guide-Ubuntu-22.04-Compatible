# Republic AI Testnet — v0.2.0 Binary Fix

> Fixes the `GetEVMCoinDenom` nil pointer dereference that caused recurring network halts.  
> Related issue: [#9](https://github.com/RepublicAI/networks/issues/9)

## What's in v0.2.0

- **Staking hooks initialization fix** — resolves the nil pointer panic on `GetEVMCoinDenom()` that crashed all nodes via gossip propagation
- **GetBlockResults gRPC endpoint** — new `GetBlockResults` and `GetLatestBlockResults` methods via forked cosmos-sdk, enables querying beginblock/endblock events over gRPC
- **WRAI chain config** and test fixes for 18-decimal chain

Release: [v0.2.0](https://github.com/RepublicAI/networks/releases/tag/v0.2.0)

---

## Update Instructions

### For Ubuntu 24.04+ Users

```bash
# 1. Stop the node
sudo systemctl stop republicd

# 2. Download the new binary
cd $HOME
wget -O republicd_new https://github.com/RepublicAI/networks/releases/download/v0.2.0/republicd-linux-amd64

# 3. Verify SHA256 checksum
echo "0788832a9f88fedfe1e2896391dd4bdb944aca85ae0632f3d30e09743a455b65  republicd_new" | sha256sum -c

# 4. Make it executable
chmod +x republicd_new

# 5. Check version
./republicd_new version

# 6. Backup old binary and replace with new one
sudo mv /usr/local/bin/republicd /usr/local/bin/republicd_v0.1.0_backup
sudo mv republicd_new /usr/local/bin/republicd

# 7. Update cosmovisor binary
cp /usr/local/bin/republicd /root/.republicd/cosmovisor/genesis/bin/republicd

# 8. Start the node
sudo systemctl start republicd

# 9. Check logs
sudo journalctl -u republicd -f -o cat --since "1 min ago"
```

---

### For Ubuntu 22.04 Users (GLIBC Fix Required)

The official binary requires GLIBC 2.38+, but Ubuntu 22.04 ships with GLIBC 2.35. We use `patchelf` to point the binary to an isolated GLIBC 2.39 without touching the system libraries.

> **First time?** You need to set up the isolated GLIBC 2.39 first. See the [full installation guide](https://github.com/coinsspor/Republic-AI-Testnet-Node-Guide) for details.

```bash
# 1. Stop the node
sudo systemctl stop republicd

# 2. Download the new binary
cd $HOME
wget -O republicd_new https://github.com/RepublicAI/networks/releases/download/v0.2.0/republicd-linux-amd64

# 3. Verify SHA256 checksum
echo "0788832a9f88fedfe1e2896391dd4bdb944aca85ae0632f3d30e09743a455b65  republicd_new" | sha256sum -c

# 4. Make it executable
chmod +x republicd_new

# 5. Patchelf — GLIBC fix for Ubuntu 22.04
patchelf --set-interpreter /opt/glibc-2.39/lib/ld-linux-x86-64.so.2 republicd_new
patchelf --set-rpath /opt/glibc-2.39/lib republicd_new

# 6. Check version
./republicd_new version

# 7. Backup old binary and replace with new one
sudo mv /usr/local/bin/republicd /usr/local/bin/republicd_v0.1.0_backup
sudo mv republicd_new /usr/local/bin/republicd

# 8. Update cosmovisor binary
cp /usr/local/bin/republicd /root/.republicd/cosmovisor/genesis/bin/republicd

# 9. Start the node
sudo systemctl start republicd

# 10. Check logs
sudo journalctl -u republicd -f -o cat --since "1 min ago"
```

---

## Post-Update Checks

```bash
# Check sync status
republicd status --home /root/.republicd 2>&1 | jq '.sync_info.latest_block_height, .sync_info.catching_up'

# Check validator signing info
republicd query slashing signing-info $(republicd comet show-validator --home /root/.republicd) --home /root/.republicd

# If jailed, unjail
republicd tx slashing unjail \
  --from wallet \
  --chain-id raitestnet_77701-1 \
  --gas auto --gas-adjustment 1.5 \
  --gas-prices 250000000arai \
  --home /root/.republicd -y
```

---

## Background

The network experienced recurring halts (~24h cycle) starting Feb 3, 2026. Root cause was a nil pointer dereference in `cosmos/evm@v0.5.1` — the EVM denom singleton (`"arai"`) was defined in genesis but never initialized at runtime. Any transaction hitting `GetEVMCoinDenom()` would panic, and since gossip propagates the TX to all nodes, every validator crashed in a panic-recover loop, blocking consensus.

Timeline:
| Event | Block | Time (UTC) |
|-------|-------|------------|
| First halt | 120,868 | Feb 3, ~04:14 |
| Recovery | 120,878 | Feb 3, ~04:40 |
| Second halt | 138,732 | Feb 4, ~06:17 |
| v0.2.0 deployed | 144,917 | Feb 4, ~14:51 |

Full technical writeup: [GitHub Issue #9](https://github.com/RepublicAI/networks/issues/9)

---

*By [@coinsspor](https://github.com/coinsspor) | [coinsspor.com](https://coinsspor.com)*
