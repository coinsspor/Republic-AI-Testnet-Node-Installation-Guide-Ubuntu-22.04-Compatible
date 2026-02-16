# Republic AI Testnet — v0.3.0 Upgrade

**Upgrade Height:** 326250  
**Upgrade Name:** v0.3.0

## Changes

- Compute validation store prefix migration (v1 -> v2)
- Restored Republic-specific defaults (home dir, bond denom)
- Updated CI release workflow for Go 1.24

---

## Ubuntu 24.04+

```bash
mkdir -p /root/.republicd/cosmovisor/upgrades/v0.3.0/bin

cd /root/.republicd/cosmovisor/upgrades/v0.3.0/bin

wget -O republicd https://github.com/RepublicAI/networks/releases/download/v0.3.0/republicd-linux-amd64

echo "bf0c88fda3ec40d8b991f87105c46ac6ddd7901d735213748de2c14e1b63a2a5  republicd" | sha256sum -c

chmod +x republicd

./republicd version
```

---

## Ubuntu 22.04 (GLIBC Fix Required)

```bash
mkdir -p /root/.republicd/cosmovisor/upgrades/v0.3.0/bin

cd /root/.republicd/cosmovisor/upgrades/v0.3.0/bin

wget -O republicd https://github.com/RepublicAI/networks/releases/download/v0.3.0/republicd-linux-amd64

echo "bf0c88fda3ec40d8b991f87105c46ac6ddd7901d735213748de2c14e1b63a2a5  republicd" | sha256sum -c

chmod +x republicd

patchelf --set-interpreter /opt/glibc-2.39/lib/ld-linux-x86-64.so.2 republicd
patchelf --set-rpath /opt/glibc-2.39/lib republicd

./republicd version
```

---

## Post-Upgrade (After Height 326250)

```bash
sudo cp /root/.republicd/cosmovisor/upgrades/v0.3.0/bin/republicd /usr/local/bin/republicd

republicd version
```

---

## Verify

```bash
republicd status 2>&1 | jq '.sync_info.latest_block_height'
```

---

*By [@coinsspor](https://github.com/coinsspor)*
