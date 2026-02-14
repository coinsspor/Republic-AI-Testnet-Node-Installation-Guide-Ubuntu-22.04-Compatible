

## Safe PEER Update (Recommended)

This script includes validation and automatic backup.

```bash
# 1. Fetch peers
PEERS=$(curl -sS https://rpc-republic-testnet.coinsspor.com/net_info | \
  jq -r '.result.peers[] | 
    select(.remote_ip | test("^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+$")) | 
    .node_info.id + "@" + .remote_ip + ":" + (.node_info.listen_addr | split(":") | last)' | \
  tr '\n' ',' | sed 's/,$//')

# 2. Check if peers found
if [ -z "$PEERS" ]; then
  echo "❌ No peers found, aborting"
  exit 1
fi

PEER_COUNT=$(echo "$PEERS" | tr ',' '\n' | wc -l)
echo "✅ Found $PEER_COUNT peers"

# 3. Backup config
cp $HOME/.republicd/config/config.toml $HOME/.republicd/config/config.toml.backup
echo "✅ Backup created: config.toml.backup"

# 4. Apply to config
awk -v peers="$PEERS" '/^persistent_peers *=/{$0="persistent_peers = \""peers"\""}1' \
  $HOME/.republicd/config/config.toml > tmp && mv tmp $HOME/.republicd/config/config.toml

# 5. Verify change
if grep -q "${PEERS:0:50}" $HOME/.republicd/config/config.toml; then
  echo "✅ Config updated successfully"
else
  echo "❌ Config update failed, restoring backup"
  cp $HOME/.republicd/config/config.toml.backup $HOME/.republicd/config/config.toml
  exit 1
fi

# 6. Restart node
sudo systemctl restart republicd
echo "✅ Node restarted"
```

---

## Rollback

If something goes wrong, restore from backup:

```bash
cp $HOME/.republicd/config/config.toml.backup $HOME/.republicd/config/config.toml && sudo systemctl restart republicd
```

