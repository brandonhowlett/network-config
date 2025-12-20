# Ubuntu Server Network Failover
### Ethernet → Static → Wi‑Fi

This repository documents a **deterministic network failover configuration** for Ubuntu Server using:

- **netplan**
- **systemd-networkd**
- **systemd (oneshot service)**

Netplan alone cannot express conditional network logic. This design fills that gap.

---

## Overview

The system enforces the following network priority order:

1. **Ethernet via DHCP (primary)**
2. **Ethernet static IP if DHCP fails**
3. **Wi‑Fi enabled only if Ethernet has no carrier**

The goal is reliable headless operation without boot hangs or race conditions.

---

## Prerequisites

- Ubuntu Server **20.04 or newer**
- `systemd-networkd` renderer
- Root or `sudo` access

---

## 1. Identify Network Interfaces

```bash
ip link
```

Example interface names used throughout this document:

- Ethernet: `enp1s0f0`
- Wi‑Fi: `wlo1`

Adjust all configurations to match your system.

---

## 2. Netplan Configuration

Create or edit the netplan configuration:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0f0:
      match:
        macaddress: "84:3a:5b:14:e3:5d"
        name: enp1s0f0
      set-name: eth0
      dhcp4: true
      optional: true
  vlans:
    vlan20:
      id: 20
      link: enp1s0f0
      dhcp4: true
      optional: true
      macaddress: "02:3a:5b:14:e3:5e"
    vlan40:
      id: 40
      link: enp1s0f0
      dhcp4: true
      optional: true
      macaddress: "02:3a:5b:14:e3:5f"
  wifis:
    wlo1:
      dhcp4: true
      optional: true
      activation-mode: manual
      access-points:
        "Sedentary Sheep":
          password: ""
```

Apply the configuration:

```bash
sudo netplan generate
sudo netplan apply
```

---

## 3. Ethernet Static Fallback (systemd-networkd)

Create a static fallback profile that **does not block boot**:

```bash
sudo nano /etc/systemd/network/10-enp1s0f0-static.network
```

```ini
[Match]
#Name=enp1s0f0
Name=eth0

[Network]
Address=10.10.30.10/16
Gateway=10.10.0.1
DNS=10.10.0.1
RequiredForOnline=no
```

Restart networkd:

```bash
sudo systemctl restart systemd-networkd
```

Verify:

```bash
networkctl status eth0 | grep Required
```

Expected output:

```
Required For Online: no
```

---

## 4. Failover Script

Create the failover script:

```bash
sudo nano /usr/local/sbin/network-failover.sh
```

```bash
#!/bin/bash

ETH="eth0"
WIFI="wlo1"
LOGTAG="network-failover"

log() {
  logger -t "$LOGTAG" "$1"
}

sleep 5

if [[ -e /sys/class/net/$ETH/carrier ]] && [[ $(cat /sys/class/net/$ETH/carrier) -eq 1 ]]; then
  log "Ethernet carrier detected. Waiting for DHCP."
  sleep 5

  if networkctl status "$ETH" | grep -q "Address:"; then
    log "Ethernet DHCP successful. Disabling Wi-Fi."
    ip link set "$WIFI" down
    exit 0
  else
    log "Ethernet DHCP failed. Applying static configuration."
    networkctl reconfigure "$ETH"
    ip link set "$WIFI" down
    exit 0
  fi
else
  log "No Ethernet carrier. Enabling Wi-Fi."
  ip link set "$WIFI" up
  networkctl reconfigure "$WIFI"
fi
```

Fix permissions and line endings:

```bash
sudo chmod 755 /usr/local/sbin/network-failover.sh
sudo sed -i 's/\r$//' /usr/local/sbin/network-failover.sh
```

Test manually:

```bash
sudo /usr/local/sbin/network-failover.sh
```

---

## 5. systemd Service

Create the service unit:

```bash
sudo nano /etc/systemd/system/network-failover.service
```

```ini
[Unit]
Description=Network failover handler
After=network.target
Wants=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/network-failover.sh

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable network-failover.service
```

Start and verify:

```bash
sudo systemctl start network-failover.service
systemctl status network-failover.service
```

Expected state:

```
Active: inactive (dead)
Status: "completed successfully"
```

---

## 6. Verification & Debugging

Logs:

```bash
journalctl -t network-failover -b
```

Interface status:

```bash
networkctl status enp1s0f0
networkctl status wlo1
```

---

## 7. Expected Behavior

| Condition                          | Result              |
|-----------------------------------|---------------------|
| Ethernet unplugged                | Wi‑Fi enabled       |
| Ethernet plugged, DHCP succeeds   | Ethernet (DHCP)     |
| Ethernet plugged, DHCP fails      | Ethernet (static)   |
| Ethernet returns                  | Wi‑Fi remains down  |

---

## 8. Common Failure Modes

### `status=203/EXEC`

Cause:
- Script not executable
- Bad or missing shebang
- CRLF line endings
- Script on a `noexec` filesystem

Fix:

```bash
chmod 755 /usr/local/sbin/network-failover.sh
sed -i 's/\r$//' /usr/local/sbin/network-failover.sh
head -n 1 /usr/local/sbin/network-failover.sh
which bash
```

---

## References

- https://netplan.io/reference
- https://www.freedesktop.org/software/systemd/man/systemd.network.html
- https://www.freedesktop.org/software/systemd/man/systemd.exec.html
