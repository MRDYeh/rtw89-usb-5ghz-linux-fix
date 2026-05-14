# TP-Link TXE70UH (RTL8852CU) Ubuntu 24.04 Complete Setup SOP — English

**Target**: TP-Link Archer TXE70UH (Realtek 8852CU, WiFi 6E USB adapter)
**System**: Ubuntu 24.04 LTS / 26.04 LTS
**Use case**: Headless server where both 2.4G and 5GHz must auto-connect on boot
**Date**: 2026-04-24 (after a 12-hour overnight debug marathon)

**Also applies to**: RTL8852BU, RTL8832CU, and other Realtek WiFi 6/6E USB adapters using the out-of-tree `rtw89` driver, where 5GHz fails with `WRONG_KEY` / `Secrets were required`.

> 中文完整版請見 [SOP.md](./SOP.md)

---

## Why 5GHz fails on rtw89 USB

**Root cause**: NetworkManager 1.46 with the `wpa_supplicant` backend auto-expands `key_mgmt` into 5 AKMs:

```
WPA-PSK WPA-PSK-SHA256 FT-PSK SAE FT-SAE
```

When the AP advertises matching capabilities, wpa_supplicant may select `WPA-PSK-SHA256`. **The rtw89 USB 8852CU driver has a bug computing PMKID under SHA256** — the MIC doesn't match the AP's, 4-way handshake M2/M3 times out, and wpa_supplicant reports the misleading `WRONG_KEY`.

2.4GHz typically only advertises basic `WPA-PSK`, sidestepping the bug.

**Why iwd backend also fails**: in theory iwd doesn't expand AKMs. **But**: NM + iwd secret passing breaks when SSIDs contain special characters (`@`, spaces, etc.), making all connections fail with `Secrets not provided`.

**v4 fix**: Run a per-interface `wpa_supplicant@IFACE.service` with your hand-written minimal config (`key_mgmt=WPA-PSK` only), **completely bypassing NM for the WiFi interface**. NM still manages ethernet and routing.

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│  enp<N> (ethernet)                   wlx<MAC> (WiFi)        │
│     │                                    │                  │
│  NetworkManager              wpa_supplicant@wlx<MAC>        │
│  (DHCP + IP + routing)    (minimal config, only WPA-PSK)    │
│                                          │                  │
│                                systemd-networkd             │
│                          (DHCP only for WiFi, [Match])      │
│                                                             │
│  NM conf.d/unmanaged-wlx.conf:                              │
│    unmanaged-devices=interface-name:wlx<MAC>                │
│  → NM doesn't touch the WiFi interface                      │
└────────────────────────────────────────────────────────────┘
```

---

## Pre-flight checks

```bash
# 1. No snap NetworkManager (the snap version fights apt NM for D-Bus)
snap list 2>/dev/null | grep network-manager && sudo snap remove network-manager

# 2. apt NM must be present and active
systemctl is-active NetworkManager || sudo apt install -y network-manager

# 3. iwd must NOT be active (leftover from a bad v3 attempt)
if systemctl is-active iwd >/dev/null 2>&1; then
  sudo systemctl disable --now iwd
  sudo systemctl mask iwd
fi
```

---

## Phase 1 — Base environment

```bash
sudo apt update
sudo apt install -y build-essential git dkms linux-headers-$(uname -r) \
                    linux-modules-extra-$(uname -r) network-manager \
                    wpasupplicant iw
```

---

## Phase 2 — Build rtw89 out-of-tree driver (via DKMS)

> **v4.1 change**: switched from `make install` to DKMS. The plain `make install` drops modules into the *current* kernel's `extra/` only — **the next kernel upgrade silently breaks WiFi** because the module isn't rebuilt against the new kernel. DKMS with `AUTOINSTALL=yes` hooks into `/etc/kernel/postinst.d/dkms`, so apt automatically rebuilds the module whenever the kernel changes. This is the root-cause fix for the recurring "kernel update broke WiFi" class of bugs (the old Troubleshooting recipe below).

```bash
cd ~
git clone https://github.com/morrownr/rtw89.git

# dkms.conf already defines PACKAGE_NAME / PACKAGE_VERSION / AUTOINSTALL=yes
PKG_VER=$(awk -F'"' '/^PACKAGE_VERSION/{print $2}' ~/rtw89/dkms.conf)
echo "rtw89 package version: $PKG_VER"

sudo cp -r ~/rtw89 /usr/src/rtw89-${PKG_VER}
sudo dkms add     -m rtw89 -v ${PKG_VER}
sudo dkms install -m rtw89 -v ${PKG_VER}

# Firmware is bundled inside morrownr/rtw89 and loaded automatically by the driver.
# If dmesg reports firmware load failure, drop it in manually:
#   sudo mkdir -p /lib/firmware/rtw89
#   sudo cp ~/rtw89/firmware/rtw8852c_fw-2.bin /lib/firmware/rtw89/
#   sudo cp ~/rtw89/firmware/rtw8852c_fw-2.bin /lib/firmware/

sudo depmod -a
```

**Verify**:

```bash
dkms status                                              # expect: rtw89/<ver>, <kernel>: installed
ls /lib/modules/$(uname -r)/updates/dkms/ | grep rtw89   # all _git modules present
grep AUTOINSTALL /usr/src/rtw89-*/dkms.conf              # = yes → next kernel auto-rebuilds
modinfo rtw89_8852cu_git | head -3                       # sanity
```

---

## Phase 3 — Driver-level workarounds

### 3a. Kernel module options (note the `_git` suffix in the module name)

```bash
sudo tee /etc/modprobe.d/rtw89.conf <<'EOF'
# The out-of-tree module names contain the _git suffix.
# v1/v2 of this SOP used plain "rtw89_core" and silently did nothing.
options rtw89_core_git disable_ps_mode=Y
EOF
```

**Verify** (after reboot):
```bash
cat /sys/module/rtw89_core_git/parameters/disable_ps_mode  # must be Y
```

### 3b. Blacklist the in-kernel rtw89 modules that would otherwise race the out-of-tree driver

```bash
sudo tee /etc/modprobe.d/blacklist-rtw89.conf <<'EOF'
blacklist rtw89core
blacklist rtw89pci
blacklist rtw_8852ce
EOF
```

### 3c. Autoload `_git` modules on boot

```bash
sudo tee /etc/modules-load.d/rtw89.conf <<'EOF'
rtw89_core_git
rtw89_usb_git
rtw89_8852cu_git
EOF
```

### 3d. Disable USB autosuspend for this specific adapter

```bash
sudo tee /etc/udev/rules.d/50-rtw89-no-autosuspend.rules <<'EOF'
# Realtek 8852CU USB: disable autosuspend. 5GHz 4-way handshake has tight timing
# and breaks if the USB device suspends between EAPOL frames.
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="35bc", ATTR{idProduct}=="0102", TEST=="power/control", ATTR{power/control}="on"
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="35bc", ATTR{idProduct}=="0102", TEST=="power/autosuspend", ATTR{power/autosuspend}="-1"
EOF
```

> For different VID:PID, adjust `idVendor`/`idProduct`. Find yours with `lsusb`.

### 3e. udev auto-modprobe fallback (belt and suspenders)

Some boot paths load the module too late. Trigger modprobe from udev when the USB device appears:

```bash
sudo tee /etc/udev/rules.d/51-rtw89-auto-load.rules <<'EOF'
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="35bc", ATTR{idProduct}=="0102", RUN+="/sbin/modprobe rtw89_8852cu_git"
EOF
sudo udevadm control --reload
```

### 3f. Rebuild initramfs so the blacklist is active from early boot

```bash
sudo update-initramfs -u
```

---

## Phase 4 — Netplan: NetworkManager as renderer only

NetworkManager must manage ethernet. Netplan just points renderer to NM. **Don't put SSIDs/passwords in netplan** — they'll conflict with nmcli profiles and spawn ghost connections.

```bash
sudo rm -f /etc/netplan/01-netcfg.yaml /etc/netplan/50-cloud-init.yaml

sudo tee /etc/netplan/00-default-nm-renderer.yaml <<'EOF'
network:
  version: 2
  renderer: NetworkManager
EOF

sudo chmod 600 /etc/netplan/*.yaml
sudo netplan generate
sudo netplan apply
```

---

## Phase 5 — 🆕 v4 core: per-interface wpa_supplicant + systemd-networkd

Replace `wlxe4fac4a5668e` with your WiFi interface name (see `ip link | grep wlx`), and fill in real SSIDs and password.

### 5a. Turn off NM's global WiFi powersave (the filename is misleading)

```bash
sudo tee /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf <<'EOF'
[connection]
# value 2 = DISABLE; value 3 = ENABLE. The filename "powersave-on" is misleading.
# Ubuntu ships this file with value 3, which destroys 5GHz handshake timing.
wifi.powersave = 2
EOF
```

### 5b. 🆕 Write a per-interface wpa_supplicant config

```bash
IFACE="wlxe4fac4a5668e"   # change this

sudo tee /etc/wpa_supplicant/wpa_supplicant-$IFACE.conf <<'EOF'
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=root
update_config=1
country=AU

# Primary: 5GHz
network={
    ssid="YOUR_5G_SSID"
    psk="YOUR_PASSWORD"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
    ieee80211w=0
    priority=10
}

# Fallback: 2.4GHz
network={
    ssid="YOUR_2.4G_SSID"
    psk="YOUR_PASSWORD"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
    ieee80211w=0
    priority=5
}
EOF
sudo chmod 600 /etc/wpa_supplicant/wpa_supplicant-$IFACE.conf
```

**Why this config works**:

- `key_mgmt=WPA-PSK` (**one** AKM, not expanded) → wpa_supplicant won't select SHA256/FT/SAE → doesn't trigger the rtw89 PMKID bug
- `ieee80211w=0` (PMF disabled) → avoids rtw89's PMF negotiation issues
- `priority=10 vs 5` → wpa_supplicant picks 5GHz when both are in range

### 5c. 🆕 Make NetworkManager ignore the WiFi interface

```bash
sudo tee /etc/NetworkManager/conf.d/unmanaged-wlxe.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:$IFACE
EOF
```

NM still manages ethernet and everything else. It just won't touch this WiFi NIC.

### 5d. 🆕 systemd-networkd handles DHCP for WiFi only

The `[Match]` clause ensures networkd only manages this one interface, avoiding any conflict with NM on ethernet.

```bash
sudo mkdir -p /etc/systemd/network
sudo tee /etc/systemd/network/99-wlxe-dhcp.network <<EOF
[Match]
Name=$IFACE

[Network]
DHCP=yes

[DHCPv4]
UseDNS=true
UseRoutes=true
RouteMetric=50
EOF

sudo systemctl unmask systemd-networkd 2>/dev/null
```

### 5e. Enable + start everything

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wpa_supplicant@$IFACE.service
sudo systemctl enable --now systemd-networkd
sudo systemctl restart NetworkManager
```

`wpa_supplicant@.service` is a systemd template unit shipped by Ubuntu. Instantiating it as `wpa_supplicant@<iface>.service` auto-uses `/etc/wpa_supplicant/wpa_supplicant-<iface>.conf`.

---

## Phase 6 — Verify (immediately)

```bash
# WiFi association
iw dev wlxe4fac4a5668e link | grep -E "SSID|freq|signal"
# Expected:
#   SSID: YOUR_5G_SSID
#   freq: 5180.0

# IP
ip addr show wlxe4fac4a5668e | grep inet

# Services
systemctl is-active wpa_supplicant@wlxe4fac4a5668e systemd-networkd

# Detailed connection log
sudo journalctl -u wpa_supplicant@wlxe4fac4a5668e --no-pager | tail -15
# Expected: "[PTK=CCMP GTK=CCMP]" and "CTRL-EVENT-CONNECTED"
```

---

## Phase 7 — Final reboot test

```bash
sudo reboot
```

After ~60 seconds, SSH back and verify:

```bash
iw dev wlxe4fac4a5668e link | grep -E "SSID|freq|signal"
sudo journalctl -u wpa_supplicant@wlxe4fac4a5668e -b --no-pager | head -15
```

**Expected**: 5GHz association within 6–15 seconds of boot.

---

## Troubleshooting

### `iw dev` returns `No such device (-19)` after reboot

USB adapter module wasn't loaded → interface not created → `wpa_supplicant@.service` failed with missing device dependency.

```bash
sudo modprobe -r rtw89_8852cu_git 2>/dev/null; sleep 2
sudo modprobe rtw89_8852cu_git
sleep 3
sudo systemctl restart wpa_supplicant@wlxe4fac4a5668e
```

If this recurs, Phase 3e's udev rule is required (it's already in the SOP above, just make sure you applied it).

### `wpa_supplicant@.service` failed with `Dependency failed`

Same as above — interface doesn't exist yet. Load the module, then **always `reset-failed` before `start`** (see next entry — `restart` alone won't recover the unit).

### `systemctl start wpa_supplicant@<IFACE>` reports success but the service stays `inactive`

**v4.1 addition.** The `wpa_supplicant@.service` template declares `Requires=sys-subsystem-net-devices-%i.device`. If the module wasn't loaded at boot, this dependency permanently fails, **and systemd does not retry the unit even after the dependency is later satisfied** — the unit is stuck in "dependency-failed" state until you explicitly clear it.

```bash
# 1. Confirm module + interface are both present now
lsmod | grep rtw89_8852cu_git
ip link | grep wlxe

# 2. Clear systemd's sticky failure record, then start
sudo systemctl reset-failed wpa_supplicant@<IFACE>.service
sudo systemctl start        wpa_supplicant@<IFACE>.service

# 3. Verify association
sudo journalctl -u wpa_supplicant@<IFACE>.service -n 20 --no-pager
# Expected: CTRL-EVENT-CONNECTED + [PTK=CCMP GTK=CCMP]
```

> This trap wasn't in v4 — only caught by hitting it in practice on a kernel-upgrade recovery. With v4 (`make install`, no DKMS), the module disappears on kernel upgrade → boot service permanently fails → even after manually `modprobe`-ing the module, the service won't recover without `reset-failed`. v4.1's DKMS Phase 2 prevents the underlying disappearance, but if you're upgrading from a v4 install on a freshly broken system, verify `dkms status` shows `installed` for the current kernel *before* rebooting.

### 5GHz connects but disconnects after a few seconds

- Check `wifi.powersave` — `cat /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf` must say `= 2`
- Check `disable_ps_mode` — `cat /sys/module/rtw89_core_git/parameters/disable_ps_mode` must be `Y`
- Check USB autosuspend — `cat /sys/bus/usb/devices/*/power/control` must include `on` (not `auto`)

### Still getting `WRONG_KEY` / `4-Way Handshake failed`

Password is confirmed correct (phone/laptop connect with same credentials)?

1. Verify `key_mgmt=WPA-PSK` in `/etc/wpa_supplicant/wpa_supplicant-<iface>.conf` — **not** `WPA-PSK WPA-PSK-SHA256` or anything expanded
2. Verify `nmcli device status` shows your WiFi interface as `unmanaged`
3. Verify router 5GHz mode is pure `WPA2-PSK (AES)` — not `WPA2/WPA3 mixed` transition mode

### Kernel update broke WiFi

**From v4.1 onward Phase 2 uses DKMS — this should self-heal** (apt's `/etc/kernel/postinst.d/dkms` hook auto-rebuilds the module against the new kernel). If WiFi is still broken after a kernel upgrade, check in order:

```bash
# 1. Did DKMS actually rebuild for the current kernel?
dkms status
# Expected: rtw89/<ver>, <current uname -r>: installed
# If the current kernel row is missing:
PKG_VER=$(awk -F'"' '/^PACKAGE_VERSION/{print $2}' /usr/src/rtw89-*/dkms.conf | head -1)
sudo dkms install -m rtw89 -v $PKG_VER

# 2. Refresh module map + load
sudo depmod -a
sudo modprobe rtw89_8852cu_git
ip link | grep wlxe   # interface should appear

# 3. Clear systemd's sticky failure on the supplicant unit, then start
sudo systemctl reset-failed wpa_supplicant@<IFACE>.service
sudo systemctl start        wpa_supplicant@<IFACE>.service

# 4. Verify
iw dev <IFACE> link
```

**Still on the v4 install path (`make install`, no DKMS)?** Switch to v4.1's DKMS install (see Phase 2) before doing anything else — otherwise this same recovery dance recurs on every kernel upgrade.

---

## Appendix A — Anti-patterns (don't do these)

- ❌ Writing SSID + password into `/etc/netplan/01-netcfg.yaml` (fights with NM profiles)
- ❌ `cron @reboot` + `systemctl restart NetworkManager` as a boot workaround (brute-force, no logs)
- ❌ `iw reg set US` manually (gets overridden by AP Country IE)
- ❌ Installing `snap install network-manager` alongside apt NM (D-Bus fight)
- ❌ `wifi.powersave = 3` (value 3 = enable, even though the filename says "powersave-on")
- ❌ Leaving `systemd-networkd` active globally (use `[Match] Name=` to scope it)
- ❌ **Using NM with wpa_supplicant backend and expecting rtw89 5GHz to work** (v2 stuck here)
- ❌ **Switching NM to iwd backend and expecting SSIDs with `@` or spaces to work** (v3 stuck here, secret passing bug)

---

## Appendix B — Root cause sanity check

If 5GHz ever fails again, verify in one minute whether it's the NM AKM-expansion bug:

```bash
# Stop the NM-managed supplicant, start a minimal one directly
sudo systemctl stop wpa_supplicant@wlxe4fac4a5668e
sudo pkill -9 wpa_supplicant

cat > /tmp/test.conf <<EOF
country=AU
network={
    ssid="YOUR_5G_SSID"
    psk="YOUR_PASSWORD"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
    ieee80211w=0
}
EOF

sudo wpa_supplicant -B -i <your_iface> -c /tmp/test.conf -D nl80211
sleep 15
iw dev <your_iface> link | grep -E "SSID|freq"
# Connects → the fix in this SOP applies (config issue, already solved)
# Still fails → driver / firmware / router — consider hardware swap (Intel AX210)

# Restore
sudo pkill -9 wpa_supplicant
sudo systemctl start wpa_supplicant@wlxe4fac4a5668e
```

---

## Appendix C — Quick health check script

Save as `~/check-wifi.sh`:

```bash
#!/bin/bash
IFACE="wlxe4fac4a5668e"   # change this

echo "=== snap NetworkManager ==="
snap list 2>/dev/null | grep network-manager && echo "⚠️ should be removed" || echo "✓ clean"

echo "=== rtw89 _git modules loaded ==="
lsmod | grep rtw89 | wc -l  # should be 4

echo "=== disable_ps_mode ==="
cat /sys/module/rtw89_core_git/parameters/disable_ps_mode  # must be Y

echo "=== USB autosuspend ==="
cat /sys/bus/usb/devices/*/power/control 2>/dev/null | sort -u | head -3  # should include 'on'

echo "=== services ==="
echo "  wpa_supplicant@$IFACE: $(systemctl is-active wpa_supplicant@$IFACE)"
echo "  systemd-networkd:     $(systemctl is-active systemd-networkd)"
echo "  NetworkManager:       $(systemctl is-active NetworkManager)"
echo "  iwd (should be masked): $(systemctl is-enabled iwd 2>&1)"

echo "=== NetworkManager should show $IFACE as unmanaged ==="
nmcli device status | grep $IFACE

echo "=== WiFi link (expect freq: 5180) ==="
iw dev $IFACE link | grep -E "SSID|freq|signal"

echo "=== IP ==="
ip addr show $IFACE | grep inet

echo "=== power_save (expect off) ==="
iw dev $IFACE get power_save
```

---

## Appendix D — How this SOP was born

Every section of this document is the result of a 12-hour overnight debug marathon chasing red herrings:

1. "It must be netplan misconfiguration." → cleaned up netplan (not the fix)
2. "It's the `fix-wifi.sh` cron brute force." → replaced with systemd (not the fix)
3. "It's snap NetworkManager fighting apt NM." → removed snap NM (fixed boot race, not 5GHz)
4. "The modprobe option isn't working." → discovered the module is actually `rtw89_core_git` not `rtw89_core` (fixed power save, not 5GHz)
5. "USB autosuspend is killing the handshake." → disabled it (helpful, not the fix)
6. "PMF / psk-flags / autoconnect-priority are wrong." → set all correctly (helpful, not the fix)
7. "`wifi.powersave = 3` is enable, not disable." → changed to 2 (helpful, not the fix)
8. "systemd-networkd is stealing the interface from NM." → masked it (helpful, not the fix)
9. "NM's iwd backend will avoid the AKM expansion." → switched to iwd — **broke all WiFi** due to secret-passing bug with SSIDs containing `@`
10. **"Let me try running wpa_supplicant directly, bypassing NM, with a minimal `WPA-PSK`-only config."** → **connected to 5GHz in 6 seconds.** Root cause confirmed.

**The actual root cause**: NetworkManager 1.46 with the wpa_supplicant backend expands `key_mgmt` into 5 AKMs (`WPA-PSK WPA-PSK-SHA256 FT-PSK SAE FT-SAE`). The rtw89 USB 8852CU driver has a bug in PMKID computation under SHA256 — the MIC doesn't match the AP's, and the 4-way handshake times out. The fix is to bypass NM for the WiFi interface entirely and run `wpa_supplicant@<iface>` with a minimal config specifying only `key_mgmt=WPA-PSK`.

This diagnosis isn't documented anywhere I could find. Google, Stack Overflow, Ask Ubuntu, GitHub issues on `morrownr/rtw89` and `lwfinger/rtw89` — nobody had put these pieces together. Hence this repo.

---

**Test environment**: Ubuntu 24.04.3 LTS, kernel 6.8.0-111-generic, TP-Link TXE70UH (USB 35bc:0102), NetworkManager 1.46.0, wpa_supplicant 2.10

**Measured boot-to-5GHz-connected**: 6 seconds on this hardware, -7 dBm signal, stable.

---

## Version history

- **v4.1 (2026-05-15)**: Phase 2 switched from `make install` to DKMS (`AUTOINSTALL=yes` makes apt's kernel postinst hook auto-rebuild the module on every kernel upgrade). Added Troubleshooting entry for the `systemctl start` succeeds but the unit stays `inactive` case — `wpa_supplicant@.service`'s `Requires=sys-subsystem-net-devices-%i.device` makes the unit permanently dependency-failed if the module wasn't loaded at boot, and `reset-failed` is required before retry. Both gaps were caught by hitting them in production after an unattended kernel upgrade.
- **v4 (2026-04-24)**: Abandoned iwd backend; switched to per-interface `wpa_supplicant@<iface>` + scoped `systemd-networkd` — completely bypass NM for the WiFi NIC. This is the actual root-cause fix for 5GHz.
- v3 (2026-04-24): Added `wifi.powersave=2`, masked systemd-networkd globally, tried iwd backend (later found to have a secret-passing bug with `@` in SSIDs).
- v2 (2026-04-24): snap NM removal, the `_git` module-name discovery, USB autosuspend off, NM connection-level hardening.
- v1 (2026-04-23): First pass, missed six critical landmines.

---

## Contributing

If this worked for you on different hardware (other RTL88XXCU / BU variants), please open an issue or PR with:

- Your adapter's `lsusb` line
- Distribution and kernel version
- Router model and 5GHz security mode
- Any tweaks you needed

Every confirmation report helps other people Google-searching their way out of the same hole.
