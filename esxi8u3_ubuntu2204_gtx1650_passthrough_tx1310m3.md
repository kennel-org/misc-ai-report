---
title: "ESXi 8.0u3 + Ubuntu 22.04 (HWE 6.8) + GTX 1650 Passthrough 成功メモ (TX1310M3)"
author: "kennel (WW) / Hana-chan (Si)"
date: "2026-01-02"
---

# ESXi 8.0u3 + Ubuntu 22.04 + TX1310M3 で GTX 1650 を認識させる手順（確定版）

目的：**ESXi 8.0u3** 上の **TX1310M3** で **GTX 1650** を PCI Passthrough し、**Ubuntu 22.04.5 (HWE 6.8)** ゲストで **`nvidia-smi` を安定して通す**。  
結論：**Ubuntu 側は `nvidia-driver-535-open`（Open Kernel Module）を使う**。Proprietary DKMS だと `RmInitAdapter failed` で死にやすい。

---

## 0. 前提（この構成で成功）

- Host: ESXi 8.0u3
- Server: FUJITSU PRIMERGY TX1310M3
- GPU: NVIDIA GeForce GTX 1650 (TU117 / 10de:1f82)
- Guest: Ubuntu 22.04.5 Desktop (Jammy) + HWE Kernel 6.8.x（例：`6.8.0-90-generic`）
- BIOS: **Above 4G Decoding = ON**
- Secure Boot: OFF（Guest 側確認：`SecureBoot disabled`）

---

## 1. ESXi Host 側（Passthrough 有効化）

1. ESXi UI → **Host > Manage > Hardware > PCI Devices**
2. 以下を **Passthrough** にする  
   1) GTX 1650（GPU function）  
   2) （任意）HDMI Audio function  
      - 今回の最終安定構成では **Audio は外して GPU 単体**にした
3. Host を **Reboot**

---

## 2. VM の基本設定（まずはここから）

1. Firmware: **EFI**
2. CPU: 2〜4 vCPU（今回 4 vCPU）
3. RAM: 8GB〜（今回 32GB）
4. **Expose hardware assisted virtualization to the guest OS = OFF**  
   - ESXi が「Nested HV not supported with PCI passthrough」警告を出すため
5. PCI Device:  
   - `NVIDIA GTX 1650` を追加（安定後に Audio 追加は検討）

---

## 3. VMX（Advanced settings / .vmx）確定値

以下は「今回の成功構成（実機ログ由来）」。

```ini
# Disable VMware virtual GPU (use passthrough GPU)
svga.present = "FALSE"

# 64-bit MMIO for BAR mapping (Above 4G + guest mem reservationが前提)
pciPassthru.use64bitMMIO = "TRUE"
pciPassthru.64bitMMIOSizeGB = "64"

# MSI can cause instability in some passthrough cases
pciPassthru0.msiEnabled = "FALSE"

# Hide hypervisor signature (anti-VM detect)
hypervisor.cpuid.v0 = "FALSE"

# Reserve all guest memory (must match your memSize)
sched.mem.min = "32768"
sched.mem.minSize = "32768"
sched.mem.pin = "TRUE"
```

> NOTE: `svga.present=FALSE` を入れると ESXi Console が基本的に見えなくなる。  
> OS インストールは SVGA 有効でも可。今回の安定構成は **SVGA 無効**で成功。

---

## 4. Ubuntu 22.04 ゲスト：ドライバ導入（成功手順）

### 4-1. まず確認（GPU が見えているか）

```bash
uname -r
lspci -nn | egrep -i 'nvidia|vga|3d|display|audio'
```

期待：
- `NVIDIA Corporation TU117 [GeForce GTX 1650] [10de:1f82]`

---

## 5. 失敗しがちなパターン（参考）

- `nvidia-driver-535`（proprietary DKMS）だと以下になりやすい  
  1) `NVRM: GPU ... RmInitAdapter failed`  
  2) `nvidia-smi` が `Killed` で落ちる（OOM ではない場合がある）

---

## 6. 解決策：Open Kernel Module に切り替える（これが本丸）

### 6-1. proprietary を消して open を入れる（必要なら）

```bash
sudo apt-get update

# If proprietary was installed, purge it (optional but recommended)
sudo apt-get -y purge 'nvidia-driver-*' 'nvidia-dkms-*' 'nvidia-kernel-*' 'nvidia-utils-*' || true
sudo apt-get -y autoremove --purge || true

# Install the open kernel module variant (535-open)
sudo apt-get install -y nvidia-driver-535-open

# Rebuild initramfs
sudo update-initramfs -u
```

### 6-2. 重要：GPU リセットのために「ESXi Power Off/On」

- `reboot`（ゲスト再起動）では GPU が完全にリセットされず、同じ失敗状態を引きずることがある
- **ESXi の UI で VM を Power Off → 10〜20 秒待つ → Power On** を推奨

---

## 7. 成功判定（この状態になれば勝ち）

### 7-1. パッケージ（-open で揃っていること）

```bash
dpkg -l | egrep "nvidia-driver-|nvidia-kernel-open|nvidia-dkms-|nvidia-utils-|linux-modules-nvidia-" || true
```

期待例：

- `nvidia-driver-535-open`
- `nvidia-dkms-535-open`
- `nvidia-utils-535`
- **混在（550/580 等）が無い**

### 7-2. モジュールが Open であること（license を見る）

```bash
modinfo nvidia | egrep -i "filename|version|license" || true
```

期待：
- `license: Dual MIT/GPL`

### 7-3. `nvidia-smi` が通る

```bash
nvidia-smi
```

期待（例）：
- GPU: `NVIDIA GeForce GTX 1650`
- VRAM: `4096MiB`
- `Driver Version: 535.274.02`
- `CUDA Version: 12.2`（ドライバ側が報告する最大互換）

---

## 8. 事故防止：ドライバを固定（hold）

```bash
sudo apt-mark hold nvidia-driver-535-open nvidia-dkms-535-open nvidia-utils-535
```

---

## 9. 追加の安定化Tips（今回効いた／効く可能性が高い）

1. **VMX 64-bit MMIO を有効化し、十分なサイズを確保**  
   1) `pciPassthru.use64bitMMIO=TRUE`  
   2) `pciPassthru.64bitMMIOSizeGB=64`
2. **MSI 無効化**  
   - `pciPassthru0.msiEnabled=FALSE`
3. **SVGA を無効化**  
   - `svga.present=FALSE`
4. **メモリ全予約**  
   - `Reserve all guest memory` 相当（pin）
5. **HDMI Audio をいったん外す**  
   - 安定後に必要なら追加検討

---

## 10. トラブルシュート（最小ログ採取）

### 10-1. まずこのワンライナー

```bash
sudo bash -lc '
set -euo pipefail
echo "=== KERNEL ==="; uname -r; echo
echo "=== PCI ==="; lspci -nn | egrep -i "vmware|svga|nvidia|vga|3d|display|audio" || true; echo
echo "=== PKG ==="; dpkg -l | egrep "nvidia-driver-|nvidia-kernel-open|nvidia-dkms-|nvidia-utils-|linux-modules-nvidia-" || true; echo
echo "=== MODINFO ==="; modinfo nvidia | egrep -i "filename|version|license" || true; echo
echo "=== LSMOD ==="; lsmod | egrep -i "nvidia|nouveau" || true; echo
echo "=== NVIDIA-SMI ==="; nvidia-smi || true; echo
echo "=== DMESG ==="; dmesg -T | egrep -i "NVRM|RmInitAdapter|GSP|GFW_BOOT|timeout|Xid|nvidia_uvm|killed|secure boot|lockdown" | tail -n 250 || true
'
```

### 10-2. 代表的な失敗ログ → 対処

1. `NVRM: ... RmInitAdapter failed!`  
   - 典型：proprietary DKMS / GPU reset 不足 / VMX設定不足  
   - 対処：**535-open へ切替 + ESXi Power Off/On + MMIO/Reserve 設定確認**

2. `nvidia-smi` が `Killed`  
   - 今回は OOM ログ無しでも発生した  
   - 対処：同上（open module で改善）

---

## 11. ここまで終わったら（次の作業の入口）

- CUDA Toolkit を入れる場合は **Toolkit のみに限定**し、`cuda` メタパッケージでドライバを上書きしないこと

---

## Appendix A. 実機ログの要点（今回の「勝ち筋」）

- proprietary 535: `RmInitAdapter failed` / `nvidia-smi` killed
- **535-open に切替後**：  
  1) `license: Dual MIT/GPL`  
  2) `nvidia_uvm` ロードOK  
  3) `nvidia-smi` OK（GTX 1650 / 4096MiB）
- VMX: `svga.present=FALSE` + `pciPassthru.use64bitMMIO=TRUE` + `pciPassthru.64bitMMIOSizeGB=64` + `sched.mem.pin=TRUE` が効いた
