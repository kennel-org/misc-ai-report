
# CUDA 12.9 + WSL 2 + Ubuntu 24.04 LTS Setup Guide
*Last updated 2025-06-11*

> **Target**: Windows 11 / Windows 10 (21H2+) with NVIDIA GPU (Pascal or newer)  
> **Goal**: Build a CUDA development environment (PyTorch / TensorFlow / C++) on WSL 2

---

## 0 . Requirements

| Item | Requirement |
| ---- | ----------- |
| **GPU** | Pascal (GTX 10xx) or later / Compute Capability ≥ 6.0 |
| **Windows** | Windows 11 or Windows 10 Build 19045+ |
| **Virtualization** | VT‑x / AMD‑V **enabled** in BIOS / UEFI |
| **Windows Features** | **Virtual Machine Platform** and **WSL** enabled |

---

## 1 . Windows side — NVIDIA driver

1. Download the **latest R535+** driver  
   - ▶︎ <https://www.nvidia.com/Download/index.aspx>
2. Install, reboot, and verify with:
   ```powershell
   nvidia-smi
   ```

---

## 2 . Install / update WSL 2

```powershell
wsl --install               # first‑time only
wsl --update                # update existing install
wsl --set-default-version 2
```

After reboot, check the service:

```powershell
sc query vmcompute          # STATE should be RUNNING
```

---

## 3 . Install Ubuntu 24.04 LTS

### 3‑1 . Via Microsoft Store  
- Microsoft Store → **Ubuntu 24.04 LTS** → **Get**

#### CLI alternative
```powershell
wsl --install -d Ubuntu-24.04
```

### 3‑2 . First‑run update
```bash
sudo apt update && sudo apt full-upgrade -y
```

---

## 4 . Install CUDA Toolkit 12.9 (inside WSL)

> **Important:** do **not** install `cuda` or `cuda-drivers` meta‑packages — they would overwrite your Windows driver.

```bash
# 4‑1 . keyring
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb

# 4‑2 . repository
echo "deb [signed-by=/usr/share/keyrings/cuda-archive-keyring.gpg] https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /" | sudo tee /etc/apt/sources.list.d/cuda-ubuntu2404.list
sudo apt update

# 4‑3 . Toolkit only
sudo apt install -y cuda-toolkit-12-9
```

### 4‑4 . Environment variables
```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

> Ubuntu 24.04 ships **GCC 14**, supported by CUDA 12.8+.  
> For older toolkits, install `gcc-13` and switch with `CC` / `CXX`.

---

## 5 . Optional libraries

```bash
# Example: cuDNN 9
sudo apt install -y cudnn9-cuda-12
```

---

## 6 . Verify installation

```bash
# GPU detection
nvidia-smi

# Build sample
cuda-install-samples-12.9.sh ~
cd ~/NVIDIA_CUDA-12.9_Samples/1_Utilities/deviceQuery
make -j$(nproc)
./deviceQuery      # should end with "Result = PASS"
```

---

## 7 . Troubleshooting

| Symptom | Cause | Fix |
| ------- | ----- | ---- |
| `HCS_E_SERVICE_NOT_AVAILABLE` / `0x80370102` | Hyper‑V feature off | Re‑enable **Virtual Machine Platform**, reboot |
| `nvcc fatal: GCC version unsupported` | CUDA ≤ 12.7 + GCC 14 | Upgrade Toolkit to 12.8+ or use GCC 13 |
| `nvidia-smi not found` | PATH unset | Add `/usr/lib/wsl/lib` to `PATH` |

---

## 8 . Recap

1. Install WSL 2 and latest NVIDIA driver on **Windows**  
2. Install **Ubuntu 24.04 LTS**  
3. Add **cuda-keyring → cuda-toolkit-12-9** (no drivers)  
4. Pass `deviceQuery` ⇒ environment ready ✅
