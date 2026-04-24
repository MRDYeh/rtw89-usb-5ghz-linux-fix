# rtw89 USB WiFi 5GHz "WRONG_KEY" Fix on Linux

> Working 5GHz auto-connect setup for Realtek **RTL8852CU / RTL8852BU** USB WiFi 6E adapters on Ubuntu 24.04+ / kernel 6.8+
> (TP-Link TXE70UH, EDIMAX EW-7822UN7, and similar)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

---

## The Problem

You bought a Realtek `8852CU` / `8852BU` USB WiFi 6E adapter. On Linux:

- ✅ **2.4GHz works fine** — auto-connects, stable
- ❌ **5GHz connection fails with `Secrets were required, but not provided`** or `WRONG_KEY`
- ❌ `wpa_supplicant` log shows: `WPA: 4-Way Handshake failed - pre-shared key may be incorrect`
- ❌ **Password is correct** — your phone/laptop connect to the same 5GHz SSID with the same password
- ❌ Router has no MAC ban, no WPA3 transition, pure WPA2-PSK (AES)
- ❌ NetworkManager + iwd backend also fails (or breaks all connections with secret-passing bug)

**You are stuck. Google finds no working answer. GitHub issues are a graveyard.**

This repo has the answer.

---

## Root Cause

**NetworkManager 1.46 + wpa_supplicant backend expands `key_mgmt` into 5 AKMs:**

```
WPA-PSK WPA-PSK-SHA256 FT-PSK SAE FT-SAE
```

When the AP advertises matching capabilities, `wpa_supplicant` may select `WPA-PSK-SHA256` or `FT-PSK` for the 4-way handshake. **The rtw89 USB driver has a bug computing PMKID under these variants** — the MIC verification fails, and `wpa_supplicant` reports the misleading `WRONG_KEY`.

2.4GHz typically only advertises basic `WPA-PSK` (SHA1), so it sidesteps the bug.

## The Fix

**Bypass NetworkManager for the WiFi interface.** Use a per-interface `wpa_supplicant@IFACE.service` with a **minimal config** that only specifies `key_mgmt=WPA-PSK` (not expanded). Let `systemd-networkd` handle DHCP via `[Match] Name=<iface>`. NetworkManager continues managing your other interfaces (ethernet, etc).

```
┌────────────────────────────────────────────────────────────┐
│  enp<N> (ethernet)                  wlx<MAC> (WiFi)         │
│     │                                   │                   │
│  NetworkManager              wpa_supplicant@wlx<MAC>        │
│  (DHCP + routing)       (minimal config, only WPA-PSK)      │
│                                         │                   │
│                               systemd-networkd              │
│                          (DHCP only for WiFi, [Match])      │
│                                                             │
│  NM conf.d/unmanaged-wlx.conf:                              │
│    unmanaged-devices=interface-name:wlx<MAC>                │
│  → NM doesn't touch the WiFi interface                      │
└────────────────────────────────────────────────────────────┘
```

After applying the full fix, boot-to-5GHz-connected is **~6 seconds** and fully automatic.

## Confirmed Working

- ✅ Ubuntu 24.04.3 LTS, kernel 6.8.0-110-generic
- ✅ TP-Link Archer TXE70UH (USB ID `35bc:0102`)
- ✅ Router: pure WPA2-PSK (AES), channel 36 (5180 MHz)
- ✅ Auto-connect on boot, -7 dBm signal, stable

## Quick Sanity Check (does this apply to you?)

If **all** of these are true, this repo's fix will solve your problem:

1. `lsusb` shows a Realtek WLAN Adapter (likely `35bc:xxxx`, `0bda:xxxx`, or similar)
2. 2.4GHz connects fine
3. 5GHz fails with `Secrets were required` / `WRONG_KEY` / 4-way handshake timeout
4. Phones / laptops connect to the same 5GHz SSID with the same password
5. You are using NetworkManager with the default `wpa_supplicant` backend (or tried `iwd` and got secret-passing errors)

## Full SOP

The complete step-by-step setup guide (in Traditional Chinese, with every command and config file) is in **[SOP.md](./SOP.md)**.

English translation TBD — contributions welcome.

## One-line Verification of the Root Cause

```bash
# Bypass NM, run a minimal wpa_supplicant config directly. If this connects 5GHz,
# the NM+wpa_supplicant AKM expansion is the root cause and the SOP fixes it.

sudo systemctl stop NetworkManager wpa_supplicant
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

sudo wpa_supplicant -B -i <your_wifi_iface> -c /tmp/test.conf -D nl80211
sleep 15
iw dev <your_wifi_iface> link | grep -E "SSID|freq"
# Connects to 5GHz (freq: 5180)? Root cause confirmed. Apply SOP.md.
# Still fails? Likely driver/firmware/router — try a different adapter (Intel AX210).

# Cleanup
sudo pkill -9 wpa_supplicant
sudo systemctl start NetworkManager
```

## Story / Why This Exists

I spent 12 hours in one night debugging this, chasing 10 different red herrings:
snap NetworkManager vs apt NetworkManager, kernel module naming (`rtw89_core_git` not `rtw89_core`), USB autosuspend, PMF negotiation, `wifi.powersave=3` (misleadingly named `powersave-on`), `systemd-networkd` vs NM, MAC bans, SSID encoding, and so on — each was a real issue, but none of them were the **5GHz-specific** root cause.

The answer (NM expanding `key_mgmt` → rtw89 SHA256 PMKID bug → handshake MIC fail) is not documented anywhere I could find. This repo exists so the next person to hit it can solve it in 10 minutes instead of a night.

If this saved you time, star the repo — it helps others find it.

## License

MIT — see [LICENSE](./LICENSE). Do whatever, just don't sue me if your router catches fire.

## Credits

- [morrownr/rtw89](https://github.com/morrownr/rtw89) — out-of-tree rtw89 driver
- Every GitHub issue commenter who tried and failed before me — your symptoms helped narrow this down

---

Contributions (especially an English translation of SOP.md, or confirmation reports from other hardware) very welcome.
