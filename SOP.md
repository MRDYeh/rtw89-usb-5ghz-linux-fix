# TP-Link TXE70UH (RTL8852CU) Ubuntu 24.04 完整安裝 SOP v4

**適用**: TP-Link Archer TXE70UH(Realtek 8852CU,WiFi 6E USB 網卡)
**系統**: Ubuntu 24.04 LTS / 26.04 LTS
**情境**: Headless server,2.4G + 5GHz 開機都要自動連
**作者**: 2026-04-24(凌晨 12 小時 marathon debug 後的真正 final 版)
**取代**: v1 / v2 / v3

---

## 版本演進

| 發現 | v1 | v2 | v3 | v4 |
|---|---|---|---|---|
| snap NM 搶 D-Bus | ❌ | ✅ | ✅ | ✅ |
| module 真名 `_git` 後綴 | ❌ | ✅ | ✅ | ✅ |
| USB autosuspend | ❌ | ✅ | ✅ | ✅ |
| `psk-flags` / `pmf` | ❌ | ✅ | ✅ | ✅ |
| netplan 只做 renderer | ❌ | ✅ | ✅ | ✅ |
| `wifi.powersave = 3`(誤導) | ❌ | ❌ | ✅ | ✅ |
| `systemd-networkd` 跟 NM 搶 | ❌ | ❌ | ✅ | ✅ |
| **v3 改 iwd backend** | | | ✅ | ❌ 有 bug |
| **v4 改 pure wpa_supplicant@** | | | | **✅ 真解** |

**v3 的 iwd 方案有 secret passing bug**(SSID 含 `@` 等特殊字元時 NM→iwd 傳 psk 會斷)。v4 完全 bypass NM 管 WiFi,用 per-interface wpa_supplicant 獨立跑。

---

## 為什麼 rtw89 USB 的 5GHz 會死

**Root cause**:NetworkManager 透過 wpa_supplicant 時,把連線的 `key_mgmt` 自動展開成 5 種 AKM:
```
WPA-PSK WPA-PSK-SHA256 FT-PSK SAE FT-SAE
```

AP(即使純 WPA2-PSK)若被認為「支援 SHA256」,wpa_supplicant 可能選到 `WPA-PSK-SHA256`。**rtw89 USB 8852CU 在處理 SHA256 PMKID 時有 bug** —— 算出來的 MIC 跟 AP 不一致 → 4-way handshake timeout → 表現為 `WRONG_KEY`。

**為什麼 2.4G 沒事**:router 2.4G 通常只 advertise basic WPA-PSK,沒觸發 bug。

**為什麼 iwd backend 也 fail**:理論上 iwd 不展開 AKM 可解,**但**:NM + iwd 整合時 SSID 含特殊字元(`@` 等)導致 NM→iwd 傳 secret 失敗 → 所有連線變 `Secrets not provided`。

**v4 解法**:直接跑 per-interface `wpa_supplicant@IFACE.service`,用你手寫的 minimal config(只 `key_mgmt=WPA-PSK`),**完全 bypass NM 管 WiFi**。NM 繼續管有線 + routing。

---

## 架構圖

```
┌────────────────────────────────────────────────────────┐
│  enp37s0 (有線)                     wlxe4fac4a5668e (WiFi)│
│     │                                   │               │
│  NetworkManager              wpa_supplicant@wlxe        │
│  (DHCP, IP, route)         (minimal config, 只 WPA-PSK)  │
│                                         │               │
│                               systemd-networkd          │
│                            (DHCP only for wlxe, [Match])│
│                                                        │
│ NM /etc/.../conf.d/unmanaged-wlxe.conf:               │
│   unmanaged-devices=interface-name:wlxe4fac4a5668e    │
│ → NM 完全不碰 WiFi interface                             │
└────────────────────────────────────────────────────────┘
```

---

## 前置檢查

```bash
# 1. 沒有 snap 版 NM
snap list 2>/dev/null | grep network-manager && sudo snap remove network-manager

# 2. apt NM 存在
systemctl is-active NetworkManager || sudo apt install -y network-manager

# 3. iwd 不該 active (v3 遺毒)
if systemctl is-active iwd >/dev/null 2>&1; then
  sudo systemctl disable --now iwd
  sudo systemctl mask iwd
fi
```

---

## Phase 1 — 基礎環境

```bash
sudo apt update
sudo apt install -y build-essential git dkms linux-headers-$(uname -r) \
                    linux-modules-extra-$(uname -r) network-manager \
                    wpasupplicant iw
```

---

## Phase 2 — 編譯 rtw89 out-of-tree driver

```bash
cd ~
git clone https://github.com/morrownr/rtw89.git
cd rtw89
make -j$(nproc)
sudo make install

sudo mkdir -p /lib/firmware/rtw89
sudo cp firmware/rtw8852c_fw-2.bin /lib/firmware/rtw89/
sudo cp firmware/rtw8852c_fw-2.bin /lib/firmware/

sudo depmod -a
```

---

## Phase 3 — Driver-level workaround

### 3a. modprobe options(module 真名含 `_git`)

```bash
sudo tee /etc/modprobe.d/rtw89.conf <<'EOF'
options rtw89_core_git disable_ps_mode=Y
EOF
```

### 3b. 黑名單內建衝突驅動

```bash
sudo tee /etc/modprobe.d/blacklist-rtw89.conf <<'EOF'
blacklist rtw89core
blacklist rtw89pci
blacklist rtw_8852ce
EOF
```

### 3c. 開機自動載入 `_git` 版

```bash
sudo tee /etc/modules-load.d/rtw89.conf <<'EOF'
rtw89_core_git
rtw89_usb_git
rtw89_8852cu_git
EOF
```

### 3d. 關 USB autosuspend

```bash
sudo tee /etc/udev/rules.d/50-rtw89-no-autosuspend.rules <<'EOF'
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="35bc", ATTR{idProduct}=="0102", TEST=="power/control", ATTR{power/control}="on"
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="35bc", ATTR{idProduct}=="0102", TEST=="power/autosuspend", ATTR{power/autosuspend}="-1"
EOF
```

### 3e. USB 網卡自動載 module(保險)

防止 module 沒自動 load 時 interface 消失:

```bash
sudo tee /etc/udev/rules.d/51-rtw89-auto-load.rules <<'EOF'
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="35bc", ATTR{idProduct}=="0102", RUN+="/sbin/modprobe rtw89_8852cu_git"
EOF
sudo udevadm control --reload
```

### 3f. 更新 initramfs

```bash
sudo update-initramfs -u
```

---

## Phase 4 — Netplan 只做 renderer(NM 不管 wlxe,但還要管 enp37s0)

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

## Phase 5 — 🆕 v4 關鍵:Pure wpa_supplicant + systemd-networkd

替換 `wlxe4fac4a5668e` 為你的網卡名,SSID / 密碼為你的。

### 5a. 關 NM 全域 WiFi powersave(檔名誤導,value 3 = enable)

```bash
sudo tee /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf <<'EOF'
[connection]
wifi.powersave = 2
EOF
```

### 5b. 🆕 寫 per-interface wpa_supplicant config

```bash
IFACE="wlxe4fac4a5668e"

sudo tee /etc/wpa_supplicant/wpa_supplicant-$IFACE.conf <<'EOF'
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=root
update_config=1
country=AU

# 5G 主力
network={
    ssid="你的_5G_SSID"
    psk="密碼"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
    ieee80211w=0
    priority=10
}

# 2.4G 備援
network={
    ssid="你的_2.4G_SSID"
    psk="密碼"
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

**為什麼這個 config 有效**:
- `key_mgmt=WPA-PSK`(**一個** AKM,不展開)→ wpa_supplicant 不選 SHA256/FT/SAE → 不觸發 rtw89 bug
- `ieee80211w=0`(PMF disabled)→ 避免 rtw89 + PMF 協商問題
- `priority=10 vs 5` → 同時有 5G / 2.4G 訊號時,wpa_supplicant 自己挑 5G

### 5c. 🆕 NM 放手 wlxe,繼續管其他 interface

```bash
sudo tee /etc/NetworkManager/conf.d/unmanaged-wlxe.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:$IFACE
EOF
```

### 5d. 🆕 systemd-networkd 只管 wlxe DHCP

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

**關鍵理解**:`[Match] Name=wlxe4fac4a5668e` 讓 networkd **只管**這張網卡,不會搶 enp37s0(那是 NM 管的)。

### 5e. 啟動所有 service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wpa_supplicant@wlxe4fac4a5668e.service
sudo systemctl enable --now systemd-networkd
sudo systemctl restart NetworkManager
```

**`wpa_supplicant@.service`** 是 Ubuntu 內建的 systemd template unit,`@` 後面的是 interface name。它會自動找 `/etc/wpa_supplicant/wpa_supplicant-<IFACE>.conf` 跑。

---

## Phase 6 — 驗證(立即)

```bash
# WiFi 連線
iw dev wlxe4fac4a5668e link | grep -E "SSID|freq|signal"
# 期望:
# SSID: 你的_5G_SSID
# freq: 5180.0

# IP
ip addr show wlxe4fac4a5668e | grep inet

# services
systemctl is-active wpa_supplicant@wlxe4fac4a5668e systemd-networkd

# 連線詳細
sudo journalctl -u wpa_supplicant@wlxe4fac4a5668e --no-pager | tail -15
# 期望: [PTK=CCMP GTK=CCMP] + CTRL-EVENT-CONNECTED
```

---

## Phase 7 — reboot 終極驗證

```bash
sudo reboot
```

等 60 秒 SSH 回來:

```bash
iw dev wlxe4fac4a5668e link | grep -E "SSID|freq|signal"
sudo journalctl -u wpa_supplicant@wlxe4fac4a5668e -b --no-pager | head -15
```

**預期**: boot 後 6-15 秒內連上 5G。

---

## Troubleshooting

### 症狀:reboot 後 `iw dev` 回 `No such device (-19)`

USB 網卡 module 沒自動 load,interface 沒 create → wpa_supplicant@.service 的 `Requires=sys-subsystem-net-devices-%i.device` dependency 失敗。

```bash
sudo modprobe -r rtw89_8852cu_git 2>/dev/null; sleep 2
sudo modprobe rtw89_8852cu_git
sleep 3
sudo systemctl restart wpa_supplicant@wlxe4fac4a5668e
```

如果常發生,加 Phase 3e 的 udev rule(已包含在 SOP 裡)。

### 症狀:`wpa_supplicant@.service` Dependency failed

Interface 不存在。先 `modprobe rtw89_8852cu_git` 救 interface,再重啟 service(見上)。

### 症狀:5G 連上但幾秒後斷

- `wifi.powersave = 3` 還沒改成 2(Phase 5a)
- `disable_ps_mode=Y` 沒生效:`cat /sys/module/rtw89_core_git/parameters/disable_ps_mode`(應是 Y)
- USB autosuspend 沒關:`cat /sys/bus/usb/devices/*/power/control`(應有 `on`)

### 症狀:連不上 5G,wpa_supplicant log 有 `WRONG_KEY` / `4-Way Handshake failed`

密碼正確的前提下:
1. 確認 `/etc/wpa_supplicant/wpa_supplicant-<IFACE>.conf` 的 `key_mgmt=WPA-PSK`(不是 `WPA-PSK WPA-PSK-SHA256`)
2. 確認沒走 NM 管 WiFi:`nmcli device status` 裡 wlxe 應該是 `unmanaged`
3. 檢查 router 5G 加密模式是 `WPA2-PSK (AES)` 純粹(不是 WPA2/WPA3 mixed)

### 症狀:kernel update 後 WiFi 掛了

out-of-tree driver 要重編:
```bash
cd ~/rtw89 && git pull && make clean && make -j$(nproc) && sudo make install && sudo depmod -a && sudo reboot
```

---

## Appendix A — 反模式(千萬不要做)

- ❌ 寫 `01-netcfg.yaml` 塞 SSID + password
- ❌ cron `@reboot` 跑 `systemctl restart NM`
- ❌ `iw reg set US` 手動設(被 AP Country IE 覆蓋)
- ❌ 只設 NM `route-metric` 不設 `autoconnect-priority`(不過 v4 根本不用 NM 管 WiFi,無關)
- ❌ 安裝 `snap install network-manager`
- ❌ `wifi.powersave = 3`(value 3 意思是 enable)
- ❌ 讓 `systemd-networkd` 管所有 interface(要用 `[Match] Name=`)
- ❌ **用 NM + wpa_supplicant backend 期待 rtw89 5GHz 能連**(v2 卡這)
- ❌ **用 NM + iwd backend 期待 SSID 含 @ 能連**(v3 卡這)

---

## Appendix B — 驗證 root cause 的 sanity check

如果將來 5G 連不上,一行確認是不是 rtw89 + NM 展開 AKM 的 bug:

```bash
# 停 NM-managed wpa_supplicant,獨立跑 minimal config
sudo systemctl stop wpa_supplicant@wlxe4fac4a5668e
sudo pkill -9 wpa_supplicant
cat > /tmp/test.conf <<EOF
country=AU
network={
    ssid="你的_5G_SSID"
    psk="密碼"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
    ieee80211w=0
}
EOF
sudo wpa_supplicant -B -i wlxe4fac4a5668e -c /tmp/test.conf -D nl80211
sleep 15
iw dev wlxe4fac4a5668e link | grep -E "SSID|freq"
# 連上 = config 問題 (回去確認 wpa_supplicant-<IFACE>.conf)
# 連不上 = driver / firmware / router

# 清理回正常狀態
sudo pkill -9 wpa_supplicant
sudo systemctl start wpa_supplicant@wlxe4fac4a5668e
```

---

## Appendix C — 懶人一鍵 check

存成 `~/check-wifi.sh`:

```bash
#!/bin/bash
IFACE="wlxe4fac4a5668e"

echo "=== snap NM ==="
snap list 2>/dev/null | grep network-manager && echo "⚠️ 該移除" || echo "✓"

echo "=== rtw89 _git modules ==="
lsmod | grep rtw89 | wc -l  # 應 4

echo "=== disable_ps_mode ==="
cat /sys/module/rtw89_core_git/parameters/disable_ps_mode  # Y

echo "=== USB autosuspend ==="
cat /sys/bus/usb/devices/*/power/control 2>/dev/null | sort -u | head -3  # 應有 on

echo "=== services ==="
echo "  wpa_supplicant@$IFACE: $(systemctl is-active wpa_supplicant@$IFACE)"
echo "  systemd-networkd:     $(systemctl is-active systemd-networkd)"
echo "  NetworkManager:       $(systemctl is-active NetworkManager)"
echo "  iwd (應 masked):      $(systemctl is-enabled iwd 2>&1)"

echo "=== NM 看 $IFACE 應該 unmanaged ==="
nmcli device status | grep $IFACE

echo "=== WiFi 連線 (期望 freq: 5180) ==="
iw dev $IFACE link | grep -E "SSID|freq|signal"

echo "=== IP ==="
ip addr show $IFACE | grep inet

echo "=== power_save (應 off) ==="
iw dev $IFACE get power_save
```

---

## Appendix D — 這份 SOP 的演進故事

這份 SOP 的每個 Phase 都是 **8 小時 debug 血淚換來**:

1. 以為是 `netplan` 亂 → 清 netplan
2. 以為是 `fix-wifi.sh` cron 暴力 → 改 systemd
3. 以為是 snap NM 衝突 → snap remove(半對,解了 boot race 但沒解 5G)
4. 以為 modprobe option 是 `rtw89_core` → 發現真名是 `rtw89_core_git`
5. 以為 USB autosuspend 是兇手 → 關掉但 5G 還 fail
6. 以為 pmf / psk-flags / priority 沒設 → 全補上但 5G 還 fail
7. 以為 `wifi.powersave = 3` 是誤導(檔名 on,值 3 是 enable)→ 改 2,5G 還 fail
8. 以為 systemd-networkd 搶 NM 是元兇 → mask,5G 還 fail
9. **以為 iwd backend 能解 AKM 展開** → 套了所有 SSID 連線炸(NM+iwd secret passing bug)
10. **血淚實驗:bypass NM 純 wpa_supplicant + minimal config → 6 秒連上 5G** ← 真解

**真 root cause**:NM 1.46 透過 wpa_supplicant backend 時,會展開 `key_mgmt` 成 5 種 AKM,rtw89 USB 8852CU 處理 SHA256 PMKID 有 bug。解法是**不要讓 NM 管這張 WiFi 卡**,per-interface 獨立跑 `wpa_supplicant@IFACE` 用你自己的 minimal config。

Global 沒人把這個寫清楚過(google 找不到),你的部落格如果公開這份 SOP 會救幾百人。

---

**測試環境**: Ubuntu 24.04.3 LTS, kernel 6.8.0-110-generic, TP-Link TXE70UH (USB 35bc:0102), NetworkManager 1.46.0, wpa_supplicant 2.10

**版本歷史**:
- **v4 (2026-04-24 06:00)**: 放棄 iwd backend,改 per-interface wpa_supplicant + systemd-networkd,完全 bypass NM for WiFi — 真正永久解 5GHz
- v3 (2026-04-24 04:00): 加 wifi.powersave=2 / systemd-networkd mask / iwd backend(iwd 方案事後證實有 bug)
- v2 (2026-04-24 02:00): snap NM、module `_git` 後綴、USB autosuspend、NM 安全欄位
- v1 (2026-04-23): 初版,6 個致命坑未涵蓋

**Boot 實測成績**:開機 6 秒自動連上 5GHz,訊號 -7 dBm,DHCP 完成。
