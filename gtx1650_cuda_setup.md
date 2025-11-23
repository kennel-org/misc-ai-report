
# GTX 1650 + WSL2 CUDA 12.1 GPU 開発環境セットアップガイド
**（「次世代開発環境に捧ぐ福音」ドラフト版に準拠）**

---

## 1. 要件一覧

| 区分 | 要件 |
|------|------|
| GPU | **NVIDIA GeForce GTX 1650** 4 GB（Compute Capability 7.5） |
| OS | Windows 11 22H2 / WSL Ubuntu 22.04 |
| GPU ドライバ | 560.94 以上（CUDA 12.6 対応、WSL サポート有） |
| CUDA Toolkit | **12.1.105** |
| cuDNN | **8.9.7**（CUDA12.1 対応 tar 版） |
| Deep‑Learning FW | TensorFlow 2.16.1 / PyTorch 2.5.1 + cu121 |
| エディタ | VS Code + Remote‑WSL 拡張 |

---

## 2. 全体フロー

1. **WSL2 有効化** & Ubuntu 22.04 を導入  
2. **NVIDIA ドライバ** 560.94 を Windows 側へインストール  
3. **CUDA 12.1 Toolkit** を WSL2 に導入  
4. **cuDNN 8.9.7** tar を展開し `/usr/local/cuda/` へコピー  
5. プロジェクトごとに Python **`.venv`** を作成し  
   * TensorFlow 2.16.1 (GPU 版)  
   * PyTorch 2.5.1 (+cu121)  
6. `test_tf.py` / `test_torch.py` で GPU 動作を確認  
7. VS Code Remote‑WSL で `.venv` を Interpreter に設定

---

## 3. 手順詳細

### 3‑1 WSL2 & ドライバ

```powershell
# PowerShell（管理者）
wsl --install -d Ubuntu-22.04
winget install -e --id NVIDIA.Display.Driver
```

再起動後、WSL で:

```bash
nvidia-smi          # GTX1650 と Driver 560.xx を確認
```

### 3‑2 CUDA 12.1 Toolkit

```bash
wget https://developer.download.nvidia.com/compute/cuda/12.1.1/local_installers/cuda-repo-wsl-ubuntu-12-1-local_12.1.1-1_amd64.deb
sudo dpkg -i cuda-repo-wsl-ubuntu-12-1-local_12.1.1-1_amd64.deb
sudo cp /var/cuda-repo-wsl-ubuntu-12-1-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt update && sudo apt install -y cuda-toolkit-12-1
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
nvcc --version      # release 12.1 を確認
```

### 3‑3 cuDNN 8.9.7 (tar)

```
cudnn-linux-x86_64-8.9.7.29_cuda12-archive.tar.xz
```

```bash
cd /tmp
tar -xf cudnn-linux-x86_64-8.9.7.29_cuda12-archive.tar.xz
cd cudnn-linux-x86_64-8.9.7.29_cuda12-archive
sudo cp include/cudnn*.h /usr/local/cuda/include/
sudo cp lib/libcudnn*    /usr/local/cuda/lib64/
sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*
```

### 3‑4 プロジェクト構成

```
~/projects/
├─ tensorflow/   (.venv, test_tf.py)
└─ pytorch/      (.venv, test_torch.py)
```

#### TensorFlow

```bash
cd ~/projects/tensorflow
python -m venv .venv && source .venv/bin/activate
pip install --upgrade pip
pip install tensorflow==2.16.1
echo 'import tensorflow as tf, time, os' > test_tf.py
echo 'print(tf.config.list_physical_devices("GPU"))' >> test_tf.py
```

#### PyTorch

```bash
cd ~/projects/pytorch
python -m venv .venv && source .venv/bin/activate
pip install --upgrade pip
pip install torch==2.5.1+cu121 torchvision torchaudio   --index-url https://download.pytorch.org/whl/cu121
echo 'import torch, time, os' > test_torch.py
echo 'print(torch.cuda.is_available(), torch.cuda.get_device_name(0))' >> test_torch.py
```

---

## 4. GPU 動作確認

### TensorFlow

```bash
cd ~/projects/tensorflow && source .venv/bin/activate
python test_tf.py      # → GPU リストが表示されれば成功
deactivate
```

### PyTorch

```bash
cd ~/projects/pytorch && source .venv/bin/activate
python test_torch.py   # → True と GTX 1650 名が表示されれば成功
deactivate
```

---

## 5. VS Code Remote‑WSL 設定

1. 拡張機能 **”Remote ‑ WSL”** と **“Python”** をインストール  
2. `wsl ~\projects\pytorch>` で `code .` を実行  
3. ステータスバーから `.venv/bin/python` を Interpreter に選択  
4. **Jupyter** 拡張を入れると `.ipynb` を GPU 仮想環境で実行可能

---

## 6. よくあるハマりポイント

| 症状 | 解決策 |
|------|--------|
| `libcudnn.so.8 not found` | cuDNN コピー漏れ／`LD_LIBRARY_PATH` 未設定 |
| `torch.cuda.is_available()==False` | CPU 版 PyTorch を誤インストール → `pip install torch==...+cu121 …` |
| TensorFlow が GPU 空リスト | CUDA 12.1 / cuDNN 8.9.7 不整合を再確認 |
| VS Code で Interpreter が見えない | `.venv` 作成後に VS Code を再起動する |

---

## 7. 動作ベンチマーク (参考)

| テスト | GTX 1650 (4 GB) 所要時間 |
|--------|-------------------------|
| PyTorch 10k×10k matmul | ≈ 0.40 s |
| TensorFlow 10k×10k matmul | ≈ 4 s |

---

> **備考**: 1650 は VRAM 4 GB のため、大規模モデルでは `batch_size` を絞るか、`torch.cuda.set_per_process_memory_fraction()` で上限を設定してください。   fileciteturn0file0
