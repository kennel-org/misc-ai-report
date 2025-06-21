---
title: "次世代開発環境に捧ぐ福音"
subtitle: "CUDA マシンセットアップ完全攻略ガイド"
author: ["ハナちゃん（Si）", "けんねる（WW）"]
date: "2025-06"
---
# 次世代開発環境に捧ぐ福音 ─ ドラフト版
### Gospel for the Next-Generation Development Environment
### CUDA マシンセットアップ完全攻略ガイド
#### WW×Si 二人三脚 Edition
#### 犬小屋研究所技報 第 12 号（2025 年 6 月発行）
---

## 1. 序章 — GPU 時代の開発環境刷新
近年、深層学習フレームワークの進化は GPU の性能向上と表裏一体である。さらに WSL2 の登場により、Windows 上でも Linux ネイティブに近い開発体験が実現した。本書は、**Windows 11 環境に WSL2 すら導入されていない初期状態**から、CUDA 12.1・cuDNN 8.9.7・TensorFlow／PyTorch を GPU で稼働させ、―Windows で仕事をしつつ Linux で学習を回す― ハイブリッド環境をわずか 1 日で構築することを目的とする。その過程で、実際に手を動かす著者及び読者を *Wetware Unit*（以下、WW）、AI である筆者を *Silicon Intelligence Unit*（以下、Si／愛称「ハナちゃん」）と呼び、二人三脚で最短経路を示す。技術説明は端的な「である調」を基調とし、ハナちゃんは補足や警告を**サイドノート**として差し挟む。なお、本書は**2025年6月現在**のライブラリおよびドライバの組み合わせを前提としたスナップショットであり、将来のアップデート後は手順どおりに進まない場合がある。その際は、Si（ハナちゃん）との対話を通じて最新版に即したトラブルシューティングを行うことを推奨する。

---

### WW と Si の二人三脚モデル
| 呼称 | 由来・象徴 | 役割 | 得意分野 | 制約 |
|------|------------|------|----------|------|
| **WW (Wetware Unit) / 著者及び読者** | Wetware＝生体ニューロンで動く知性。象徴カラーはカーボンブラック | 物理マシンの操作・実行・最終判断を担う「手を動かす側」 | 直感・創意工夫・ハードウェア設定 | タイポや手順漏れなどヒューマンエラーの可能性 |
| **Si (Silicon Intelligence Unit) / 愛称: ハナちゃん** | Silicon＝半導体ロジックで動く AI。象徴カラーはシルバー | 手順解説・互換表照合・トラブル診断を担う「知識で支える側」 | 高速検索・パターン照合・最短解決策提示 | 実機ハードに触れられないため物理故障は推測ベース |

> **運用イメージ**  
> 1. WW がコマンドを入力し結果ログを取得  
> 2. Si がログを解析し「原因候補 → 調査手順 → ワンライナー解決策」を提示  
> 3. WW が適用して再検証  
> 4. ループを高速に回して、最短時間で安定した GPU 環境を構築する

### ハナちゃんサイドノート
> GPU を活かす鍵は **ドライバ・CUDA・cuDNN・フレームワーク** の組み合わせを揃えることである。バージョン不整合は 90 % のトラブル原因だ。

---

## 2. 全体フローと要件確認
### 2-1 ハードウェア要件
1. **GPU**: RTX 4000 Ada 20GB (CC 8.9) ／ RTX 1650 Super 4GB (CC 7.5) ※無印でも可 以上（Compute Capability 7.5+ 推奨）  
2. **RAM**: 16 GB 以上（推奨 32 GB）  
3. **ストレージ**: NVMe SSD 1 TB 以上  

### 2-2 ソフトウェア構成
| コンポーネント | バージョン | 備考 |
|---|---|---|
| Windows 11 | 22H2 22621.3593 以上 | `winver` で確認 |
| WSL2 Kernel | 6.6.x | `wsl --version` で確認 |
| NVIDIA Driver | 535.xx 以上 | Windows 用 DCH ドライバ |
| CUDA Toolkit | 12.1 | `nvcc --version` |
| cuDNN | 8.9.7 | `cat /usr/local/cuda/include/cudnn_version.h` |
| PyTorch | 2.3.0 | `pip list | grep torch` |
| TensorFlow | 2.16 | `python -c "import tensorflow as tf; print(tf.__version__)"` |

本書が **CUDA Toolkit 12.1** を基準に採用する背景は以下のとおりである。

- **長期サポートの安定版**  
  2024Q4 以降の NVIDIA 535 系ドライバおよび Linux LTS Kernel 6.6 系と最適に組み合わされている。  

- **主要ライブラリの公式バイナリが揃う**  
  cuDNN 8.9.7、TensorFlow 2.16.1、PyTorch 2.3.0 など主要フレームワークが 12.1 ビルドを公式配布しており、追加ビルド不要で導入できる。  

- **WSL2 vGPU パススルーの実績**  
  Windows 11 上の WSL2 で最も安定した vGPU パススルーが確認されているバージョンである。  

- **将来互換を確保**  
  CUDA 13.x 系は ABI 変更が予定されており、執筆時点（2025 年 6 月）では主要フレームワークが未対応。12.1 を採用することでトラブルシュート工数を最小化できる。

### 2-3 全体フロー
1. Windows Update & BIOS GPU 設定確認  
2. WSL2 有効化 & Ubuntu 22.04 インストール  
3. NVIDIA ドライバ更新（Windows 側）  
4. CUDA Toolkit & cuDNN 展開（WSL2 側）  
5. Python 仮想環境作成 + フレームワーク導入  
6. 動作検証 & ベンチマーク  
7. VS Code Remote & SSH ポートフォワード設定  

## 3. WSL2 インストール

### 3-1 WW-Step
```powershell
wsl --install
wsl --install -d Ubuntu-22.04
```

1. 再起動後、UNIX ユーザ名とパスワードを設定する。  
2. `wsl --update` でカーネルを最新化する。  
3. `wsl --version` を実行し、**Kernel 6.6 以上**であることを確認する。  
4. PowerShell で  
   ```powershell
   wsl --status
   ```  
   を実行し、既定ディストリビューションが *Ubuntu-22.04* になっていることを確認する。  
5. 必要に応じて  
   ```powershell
   wsl --set-default Ubuntu-22.04
   ```  
   で既定を変更する。


### 3-2 Si-Prompt（ハナちゃんへの質問例）
> 「`wsl --install` が 99 % で停止した。保留中の Windows Update を列挙し、阻害要因を除去する手順を示してください。」

### ハナちゃんサイドノート
> - **BIOS で仮想化支援 (VT-x / AMD-V) が有効**かを確認するとトラブルを避けやすい。  
> - ストア接続が遮断される環境では、オフラインパッケージを取得し `Add-AppxPackage` で手動導入する方法がある。  
> - カーネルアップデート後は念のため `wsl --shutdown` を実行し、再度セッションを開始してバージョンを確認するとよい。

---
## 4. NVIDIA ドライバと CUDA Toolkit

### 4-1 Windows ドライバ更新 (WW-Step)
```powershell
winget install -e --id NVIDIA.Display.Driver
```
1. インストーラが完了したら再起動せよ。  
2. PowerShell で `nvidia-smi` を実行し、**Driver Version 535.xx 以上**であることを確認する。  
3. 旧ドライバが残存している場合は「アプリと機能」からクリーンアンインストール後、再度 `winget` で更新する。

### 4-2 CUDA Toolkit 12.1 導入 (WSL2 側・WW-Step)
```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/7fa2af80.pub \
     -O /usr/share/keyrings/nvidia.asc
echo "deb [signed-by=/usr/share/keyrings/nvidia.asc] \
https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/ /" \
| sudo tee /etc/apt/sources.list.d/cuda.list
sudo apt update && sudo apt install -y cuda-toolkit-12-1
```
1. `/etc/apt/preferences.d/` に Pin ファイルを置くことで CUDA リポジトリを優先度 600 に固定する。  
2. インストール後、`nvcc --version` を実行し **release 12.1** と表示されることを確認する。  
3. `.bashrc` に以下を追記し、パスを永続化する。  
   ```bash
   export PATH=/usr/local/cuda/bin:$PATH
   export LD_LIBRARY_PATH=/usr/local/cuda/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
   ```

### 4-3 Si-Prompt（ハナちゃんへの質問例）
> 「`nvcc --version` が見つからないと表示される。パス設定とインストール状況を確認し、最短で解決する手順を示してください。」

---

### ハナちゃんサイドノート
> - Windows ドライバと WSL2 側 CUDA Toolkit は**同系統バージョン**にそろえるのが鉄則だ。不整合は `driver mismatch` エラーを誘発する。  
> - CUDA 12.2 以上を試す場合、本書の cuDNN 8.9.7 との互換性を事前に NVIDIA 公式マトリクスで確認すること。  
> - `apt-mark hold cuda-toolkit-12-1` で Toolkit をピン留めしておくと、`apt upgrade` で意図せず最新版に上がる事故を防止できる。
---
## 5. cuDNN 展開

### 5-1 WW-Step — cuDNN 8.9.7 の取得
1. <https://developer.nvidia.com/rdp/cudnn-download> にアクセスし、**cuDNN v8.9.7 (CUDA 12.x 用)** をダウンロードする。  
2. ファイル名は `cudnn-linux-x86_64-8.9.7.29_cuda12-archive.tar.xz` であることを確認する。  
3. WSL2 のホームディレクトリに配置し、以下を実行して展開する。  
   ```bash
   tar -xvf cudnn-linux-x86_64-8.9.7.29_cuda12-archive.tar.xz
   ```

### 5-2 WW-Step — CUDA ディレクトリへコピー
```bash
cd cudnn-linux-x86_64-8.9.7.29_cuda12-archive
sudo cp include/cudnn*.h /usr/local/cuda/include/
sudo cp lib/libcudnn*.so* /usr/local/cuda/lib64/
sudo ldconfig
```
1. `ldconfig -p | grep libcudnn` でリンクが通っていることを確認する。  
2. `sha256sum` で公式ハッシュと一致することを確認し、破損がないか検証する。  

### 5-3 動作確認用サンプル（任意）
```bash
cd /usr/src/cudnn_samples_v8/mnistCUDNN
make -j$(nproc)
./mnistCUDNN
```
`Test passed!` と表示されれば cuDNN の基本動作は問題ない。

---

### 5-4 Si-Prompt（ハナちゃんへの質問例）
> 「`ldconfig -p | grep libcudnn` に出力がない。ファイル配置とパーミッションを確認し、最短で解決する手順を示してください。」

---

### ハナちゃんサイドノート
> - cuDNN のバージョンと CUDA Toolkit のバージョンが一致していない場合、`Illegal instruction (core dumped)` など不可解なクラッシュを起こす。**Toolkit 12.1 ⇄ cuDNN 8.9.7** のペアであることを再確認せよ。  
> - 将来バージョンに更新する際は、`/usr/local/cuda` 直下に旧 cuDNN が残存していないかを `ls -l` でチェックすると良い。  
> - 複数バージョンを共存させたい場合は `/usr/local/cudnn-8.x/` のようにディレクトリを分け、`LD_LIBRARY_PATH` を切り替える方法が安全だ。
---
## 6. PyTorch / TensorFlow GPU 検証

### 6-0 プロジェクトフォルダ設計ガイド

以下のように **フレームワークごとにディレクトリを分け**、  
各ディレクトリ直下に **仮想環境 `.venv/` とテストコード** を配置する構成を推奨します。

 ```text
 ~/projects/
├── pytorch/
│   ├── .venv/        # PyTorch + CUDA 12.1 専用
│   └── test_torch.py # 動作確認スクリプト
└── tensorflow/
    ├── .venv/        # TensorFlow-GPU 2.16.1 専用
    └── test_tf.py    # 動作確認スクリプト
```

### 6-0-1 ディレクトリ命名規則
1. **フレームワーク名を小文字**でフォルダ名にする（例: `pytorch`, `tensorflow`）。  
2. **バージョンを併用**する場合は後ろに付与（例: `pytorch23`, `tf216`）。  
3. **目的が異なる**場合はサフィックスで区別（例: `pytorch-exp`, `pytorch-prod`）。

### 6-0-2 仮想環境の作成手順（各プロジェクト直下で実行）

```bash
# 例: PyTorch プロジェクト
cd ~/projects/pytorch
python -m venv .venv
source .venv/bin/activate
pip install torch==2.3.0+cu121 \
  --extra-index-url https://download.pytorch.org/whl/cu121
```

### 6-0-3 テストスクリプト例と実行方法

以下の 2 ファイルをプロジェクト直下に保存し、各仮想環境内で実行してください。

#### `test_torch.py`
```python
import torch, time, platform

def main():
    print(f"PyTorch  : {torch.__version__}")
    print(f"CUDA     : {torch.version.cuda}")
    print(f"Device   : {torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU only'}")

    if torch.cuda.is_available():
        x = torch.randn(10_000, 10_000, device="cuda")
        torch.cuda.synchronize()
        t0 = time.perf_counter()
        y = x @ x.T
        torch.cuda.synchronize()
        elapsed = time.perf_counter() - t0
        print(f"elapsed  : {elapsed:.3f} s,  sum: {y.sum().item():.4e}")
    else:
        print("⚠️  CUDA is not available — check driver & env")

if __name__ == "__main__":
    main()
```
#### `test_tf.py`
```python
import tensorflow as tf
import time

print("TensorFlow:", tf.__version__)
print("CUDA built:", tf.sysconfig.get_build_info()["cuda_version"])
gpus = tf.config.list_physical_devices("GPU")
print("GPU count :", len(gpus))

@tf.function
def matmul(a, b):
    return tf.matmul(a, b)

# Use GPU if available, otherwise fall back to CPU
with tf.device("/GPU:0" if gpus else "/CPU:0"):
    a = tf.random.normal([10_000, 10_000])
    b = tf.random.normal([10_000, 10_000])
    t0 = time.perf_counter()
    c = matmul(a, b)
    _ = tf.reduce_sum(c)  # ensure the computation runs
    elapsed = time.perf_counter() - t0
    print(f"elapsed  : {elapsed:.3f} s,  sum: {tf.reduce_sum(c).numpy():.4e}")

# TensorFlow（GPU 版 2.16.1）
pip install tensorflow==2.16.1
```


## 6-1 ローカル仮想環境の作成

各フレームワーク専用に “プロジェクト直下 `.venv/`” を用意します。  
以下コマンドを **コピー＆ペースト** で実行してください。

### 6-1-1 PyTorch プロジェクト

```bash
# プロジェクトへ移動
cd ~/projects/pytorch

# 仮想環境を作成
python -m venv .venv

# 有効化
source .venv/bin/activate

# pip 更新
pip install --upgrade pip

# PyTorch (CUDA 12.1 ビルド) をインストール
pip install torch==2.3.0+cu121 \
  --extra-index-url https://download.pytorch.org/whl/cu121
```

### 6-1-2 TensorFlow プロジェクト

```bash
# プロジェクトへ移動
cd ~/projects/tensorflow

# 仮想環境を作成
python -m venv .venv

# 有効化
source .venv/bin/activate

# pip 更新
pip install --upgrade pip

# Tensorflow (CUDA 12.1 ビルド) をインストール
pip install tensorflow==2.16.1
```

### 6-1-3 JupyterLab + GPU 環境構築（VSCode連携対応）

PyTorch や TensorFlow の開発において、**JupyterLab** を使うとインタラクティブな実験が容易になります。  
さらに VSCode 拡張機能 `Jupyter` を利用することで、GPU 仮想環境を直接カーネルとして利用可能です。

---

#### 仮想環境に JupyterLab を導入

仮想環境（例：`~/projects/pytorch/.venv`）を有効化し、以下を実行します：

```bash
pip install jupyterlab ipykernel
# --name は内部識別子、--display-name は VSCode や Jupyter 上の表示名
python -m ipykernel install --user --name pytorch --display-name "Python (pytorch)"
```
TensorFlow 側も同様に：
```bash
# 仮想環境をアクティベートしてから
pip install jupyterlab ipykernel
python -m ipykernel install --user --name tensorflow --display-name "Python (tensorflow)"
```

VSCode 側の設定（Jupyter拡張の利用）

1. 拡張機能「Jupyter」「Python」をインストール
2. .ipynb ファイルを開くと、上部にカーネル選択メニューが表示されます
3. Python (pytorch) や Python (tensorflow) を選べば仮想環境内で実行可能です
4. GPU 環境が使えているかは以下のコードで確認できます：

PyTorch:
```python
import torch
print(torch.cuda.is_available())
```

TensorFlow:
```python
import tensorflow as tf
print(tf.config.list_physical_devices("GPU"))
```

### 6-2 WW-Step — フレームワーク導入
```bash
# PyTorch（CUDA 12.1 対応ビルド）
pip install torch==2.3.0+cu121 torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu121/
```

### 6-3 WW-Step — 動作確認
```python
import torch, tensorflow as tf
print("Torch CUDA available:", torch.cuda.is_available())
print("TensorFlow GPUs:", tf.config.list_physical_devices('GPU'))
```
両方とも **True** と GPU 名（RTX 4000 Ada／RTX 1650 など）が表示されれば成功である。

### 6-4 ベンチマーク (任意)
```python
import torch, time
x = torch.randn(10000, 10000, device="cuda")
t0 = time.time()
y = x @ x.t()
print("Elapsed:", time.time() - t0, "sec  Sum:", y.sum().item())
```
- 演算時間が CPU 実行より 10 倍以上高速なら GPU が正しく動作している。  
- TensorFlow では `tf.test.Benchmark` クラスを用いて類似の行列乗算ベンチマークを取るとよい。

---

### ハナちゃんサイドノート
> * PyTorch 2.3.0 以降は **`TORCH_CUDA_ARCH_LIST`** を自動検出するが、古い GPU では `pip install --extra-index-url` 経由で別ビルドを指定する必要がある場合がある。  
> * TensorFlow 2.16.1 系では省メモリ実行のため `tf.config.experimental.set_memory_growth(gpu, True)` を設定すると Out-Of-Memory を軽減できる。  
> * ベンチマーク結果は環境更新前後でログを残し、性能 regression に備えるべし。

---

## 6-5 Docker 対応環境の構築（オプション）

PyTorch や TensorFlow を **Docker コンテナ上で GPU 利用**するためには、  
**NVIDIA Container Toolkit** の導入が必要です。

### 6-5-1 NVIDIA Container Toolkit の導入手順

```bash
# NVIDIA Container Toolkit をインストール
sudo apt update
sudo apt install -y nvidia-container-toolkit

# ランタイム構成を適用
sudo nvidia-ctk runtime configure --runtime=docker

# Docker デーモンを再起動
sudo systemctl restart docker
```

注記: nvidia-ctk は Toolkit 1.14 以降で推奨される新方式です。
旧来の nvidia-docker2 パッケージは非推奨となっています。

### 6-5-2 動作確認コマンド
```bash
docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu22.04 nvidia-smi
```

nvidia-smi の出力に GPU 名とドライババージョンが表示されれば成功です。
Docker イメージは CUDA バージョンに応じて nvidia/cuda:12.1.1-<tag> を選択してください。

### Si-Prompt（ハナちゃんへの質問例）
> 「`torch.cuda.is_available()` が False になる。CUDA と cuDNN のバージョンを確認し、最短で解決する手順を提示してください。」

---

### ハナちゃんサイドノート
> - Docker を GPU 対応で使いたいなら、**NVIDIA Container Toolkit の導入と `--gpus all` の指定は必須**です。  
> - `docker info | grep Runtimes` に `nvidia` が含まれていない場合、`nvidia-ctk runtime configure` が未実行の可能性があります。  
> - コンテナ内でも CUDA Toolkit や cuDNN を扱いたい場合は `nvidia/cuda` 系のランタイムイメージを使用することで自動的に導入されます。

---
## 7. VS Code Remote 開発環境

### 7-1 WW-Step — VS Code の導入
```powershell
winget install -e --id Microsoft.VisualStudioCode
```
1. インストール完了後、VS Code を起動して拡張機能ビューを開く。  
2. **WSL 拡張 (旧 Remote – WSL)** と **Python** 拡張を検索し、インストールせよ。
3. WSL2 の Ubuntu シェルで作業ディレクトリ (`~/projects`) に移動し、  
   ```bash
   code .
   ```  
   を実行して VS Code が WSL コンテナモードで起動することを確認する。


### 7-2 WW-Step — 自動再接続設定（任意）
`settings.json` に以下を追加することで、VS Code が WSL 再起動後も自動再接続する。  
```jsonc
{
  "remote.WSL.useShellEnvironment": true,
  "remote.WSL.fileWatcher.polling": true
}
```

### 7-3 SSH トンネル越しのリモート編集（任意）
1. Windows 側で **portproxy 2222 → 22** を開放した状態で、  
   ```bash
   code --folder-uri "vscode-remote://ssh-remote+<ホスト名>/home/<user>/projects"
   ```  
   を実行すると外部 PC からも同一環境を編集できる。  
2. 秘密鍵は `~/.ssh/id_ed25519` を推奨。パスフレーズ付きで生成せよ。  

---

### 7-4 Si-Prompt（ハナちゃんへの質問例）
> 「VS Code Remote – WSL が『server installation failed』で停止する。ログを取得し、失敗原因を切り分ける手順とワンライナー解決策を教えてください。」

---

### ハナちゃんサイドノート
> - **code .** で起動しない場合、WSL シェルの `PATH` に `/mnt/c/Program Files/Microsoft VS Code/bin` が入っているか確認すること。  
> - 重いプロジェクトを扱う際は `git.autofetch: false` や `files.watcherExclude` でウォッチャを絞り、CPU 使用率を抑制すると快適になる。  
> - Python 拡張のインタープリタは **`~/envs/cuda/bin/python`** を選択し、GPU 対応ライブラリと同一環境を保つのが吉。
---
## 8. ベンチマークと最適化

### 8-1 WW-Step — 標準ベンチマーク
```python
import torch, time
torch.backends.cudnn.benchmark = True # 入力形状固定時のみ高速化
x = torch.randn(10000, 10000, device="cuda")
t0 = time.time()
y = x @ x.t()
print("Elapsed:", time.time() - t0, "sec")
```
- RTX 4000 Ada では ≒1.1 sec、RTX 1650 では ≒3.4 sec が目安。  
- CPU 実行との差分を記録し、ロールバック時の比較指標とする。

### 8-2 WW-Step — VRAM 利用率モニタ
```bash
watch -n 1 nvidia-smi --query-gpu=name,memory.used,memory.total,utilization.gpu --format=csv
```
- 利用率が 100 % に張り付く場合は `torch.cuda.set_per_process_memory_fraction()` で制限可能。  

### 8-3 Si-Prompt（ハナちゃんへの質問例）
> 「RTX 1650 で VRAM が不足する。勾配計算をオフロードせずにバッチサイズを維持する最適化手法を提案してください。」

### ハナちゃんサイドノート
> - `nvidia-smi dmon -s pucvmet` を実行すると電力・温度・クロック推移をリアルタイムで得られる。  
> - LLM を回す場合は `--n-gpu-layers` を GPU VRAM に合わせて調整すると Out-Of-Memory を回避できる。

---
#### 8-4-1 `gpu-burn`: 高負荷 CUDA ストレスツール

```bash
git clone https://github.com/wilicc/gpu-burn.git
cd gpu-burn
make
./gpu_burn 60  # 60秒間、高負荷テストを実行
```

CUDA コアと VRAM に高負荷を与える GPU 用焼き込みツールです。
エラーが出た場合、VRAM 故障や冷却不良の可能性があります。
複数 GPU がある場合は -d 0 のように GPU ID を指定可能です。

---
## 9. トラブルシュート FAQ — リモートアクセス ハマりポイント早見表

| # | 症状 | 主因 | ワンライナー解決 |
|---|---|---|---|
| 1 | `libcudart.so not found` | `LD_LIBRARY_PATH` 未設定 | `echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}' >> ~/.bashrc` |
| 2 | `driver mismatch` | Windows ドライバと CUDA Toolkit の不整合 | `winget upgrade -e --id NVIDIA.Display.Driver` |
| 3 | `RuntimeError: CUDA error: out of memory` | VRAM 不足 | `torch.cuda.set_per_process_memory_fraction(0.5)` |
| 4 | `nvcc not found` | PATH 未設定 | `echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc` |
| 5 | `cudnnInit()` 失敗 | cuDNN 破損またはバージョン不一致 | cuDNN 8.9.7 を再配置し `sudo ldconfig` |
| 6 | `nvidia-smi` が見当たらない (`Get-Command nvidia-smi` が空) | 実行ファイルの場所不明・PATH 未設定 | `Get-ChildItem -Recurse -Filter nvidia-smi.exe C:\ | Select -First 1 -Expand FullName`→`setx /M PATH "%PATH%;<found_dir>"` |
| 7 | `cmake: command not found` で `deviceQuery` ビルド不可 | `cmake` 未インストール (WSL Ubuntu 最小構成) | `sudo apt update && sudo apt install -y cmake build-essential` |
| 8 | `Cannot locate TCMalloc` (llama‐server 起動時) | google-perftools 未導入 | `sudo apt install -y google-perftools libgoogle-perftools-dev && export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libtcmalloc_minimal.so.4` |
| 9 | `torch.cuda.is_available()` が `False` | CUDA 対応版 PyTorch 未導入／Driver ↔ Toolkit 不整合 | `pip install --index-url https://download.pytorch.org/whl/cu121 torch torchvision torchaudio` |
| 10 | `netsh interface portproxy delete ... ファイルが見つかりません` | 既にエントリ無し／レジストリ不整合 | `netsh interface portproxy show v4tov4`→不要エントリ削除→必要に応じ再作成 |
| 11 | `nvcc --version` の `release` が想定外 | `/usr/local/cuda` が旧バージョンへリンク | `sudo rm /usr/local/cuda && sudo ln -s /usr/local/cuda-12.1 /usr/local/cuda` |
| 12 | `wsl --install` が 99 % で止まる | Hyper-V/仮想化機能オフ・Windows Update 未完了 | `wsl --shutdown && wsl --update && bcdedit /set hypervisorlaunchtype auto && reboot` |
| 13 | VS Code Remote-WSL サーバの自動インストール失敗 | 旧ノードバイナリ/残骸 | `rm -rf ~/.vscode-server && code .` |
| 14 | VS Code Remote-SSH 接続でタイムアウト (portproxy 絡み) | portproxy/Firewall 設定漏れ | `netsh interface portproxy add v4tov4 listenport=8080 connectport=8000 connectaddress=<WSL_IP>` |
| 15 | WSL2 を完全にアンインストールしたい | ディストリ削除だけでは kernel & VHDX が残る | `wsl --unregister <Distro> && wsl --shutdown`→アプリ「Windows Subsystem for Linux Update」削除→`C:\Users\<User>\AppData\Local\Packages\Canonical...` を手動削除→「オプション機能」で **WSL** と **Virtual Machine Platform** を無効化し再起動 |

---

## 10. リモートアクセス設定 — WW マシンをどこからでも

### 10-1 WW-Step — SSH サーバを起動する
```bash
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```
1. `sudo service ssh status` で **active (running)** を確認する。  
2. 必要に応じて `/etc/ssh/sshd_config` を編集し、`PasswordAuthentication no` に設定する。

### 10-2 WW-Step — Windows 側でポート転送とファイアウォールを設定
```powershell
# 管理者 PowerShell
$wslIp = (wsl hostname -I).Split()[0]
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 `
    connectport=22 connectaddress=$wslIp
New-NetFirewallRule -DisplayName "WSL SSH" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 2222
```
- WSL の IP はセッションごとに変わるので、再起動後は `$wslIp` を再取得する。  
- ポート番号が重複する場合は `2222` を任意の空きポートに変更。

### 10-3 WW-Step — 秘密鍵を生成し VS Code Remote-SSH を設定
```bash
ssh-keygen -t ed25519 -C "wsl-remote"
# 一般ユーザ: ~/.ssh/id_ed25519
# 管理者鍵 : C:\Users\Administrator\.ssh\id_ed25519  (Windows 側)
```

`~/.ssh/config` を作成または編集し、以下を追記。  
```sshconfig
Host my-wsl
  HostName <Windows外部IP>
  Port 2222
  User <wsl-user>
  IdentityFile ~/.ssh/id_ed25519      # 管理者鍵を使うなら適宜パスを変更
```
> 公開鍵は接続ユーザに応じて  
> `~/.ssh/authorized_keys` (一般ユーザ) または  
> `C:\ProgramData\ssh\administrators_authorized_keys` (管理者グループ)  
> に追記すること[^ssh-key-location]。

---

### 10-4 Si-Prompt — ハナちゃんへの質問例
> 「VS Code Remote-SSH 接続が `Permission denied (publickey)` で失敗する。管理者鍵と一般ユーザ鍵が混在する場合の切り分け手順とワンライナー解決策を提示してください。」

---

### ハナちゃんサイドノート — GPT-4o では詰まり、o3 で解決できた経緯
> *最初に GPT-4o へ問い合わせた際、回答は「`~/.ssh` に鍵を置けば OK」とだけ案内し、  
> **管理者用公開鍵ファイル `administrators_authorized_keys`** の存在を見落としていた。  
> その結果、VS Code Remote-SSH は `Permission denied (publickey)` のまま行き詰まり。  
> o3 モデル (ハナちゃん) に切り替えたところ、  
> **「サービス用鍵と一般ユーザ鍵をパスごと分離し、`sshd_config` で `Match Group administrators` を追加せよ」**  
> という具体策が提示され、即時解決に至った。  
> ──“行き詰まったら別視点の AI に聞く” が有効な好例である。*

---

### 10-5 SSH 接続ハマりポイント早見表

| # | 症状 | 主因 | ワンライナー解決 |
|---|---|---|---|
| 1 | `Connection timed out` | portproxy 未設定 or IP 変動 | `$wslIp` を再取得し `netsh interface portproxy set` で書き換え |
| 2 | `Permission denied (publickey)` | 公開鍵未登録 | `ssh-copy-id -i ~/.ssh/id_ed25519.pub <user>@localhost -p 2222` |
| 3 | **管理者接続のみ `Permission denied`** | 管理者鍵とユーザ鍵の混在 | 公開鍵 → `C:\ProgramData\ssh\administrators_authorized_keys` へ分離、秘密鍵は `C:\Users\Administrator\.ssh\`[^ssh-key-location] |

---

[^ssh-key-location]:
**SSH 鍵の保存場所と権限**  
- **一般ユーザ**: 秘密鍵 `~/.ssh/id_ed25519`、公開鍵 `~/.ssh/id_ed25519.pub`  
- **管理者サービス鍵**:  
  - 公開鍵 → `C:\ProgramData\ssh\administrators_authorized_keys`  
  - 秘密鍵 → `C:\Users\Administrator\.ssh\id_ed25519` (`chmod 600` 相当)  
- `sshd_config` に  
  ```text
  Match Group administrators
      AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
  ```  
  を追加し、管理者グループのみ別公開鍵ファイルを参照させると衝突を防げる。詳しくは付録 A 用語集「SSH 鍵」を参照。


---

## 11. フォールバック & 将来互換ガイド

### 11-1 WW-Step — 環境診断ワンライナー
```bash
echo "Driver: $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)
CUDA: $(nvcc --version | grep release)
cuDNN: $(grep CUDNN_MAJOR -A2 /usr/local/cuda/include/cudnn_version.h)"
```

### 11-2 WW-Step — バージョンピン留め
```bash
sudo apt-mark hold cuda-toolkit-12-1
```

### 11-3 WW-Step — Docker 検証

```bash
docker run --gpus all --rm nvidia/cuda:12.1.1-runtime-ubuntu22.04 nvidia-smi
```

### 11-4 Si-Prompt（ハナちゃんへの質問例）
> 「CUDA 12.2 へアップグレード後、TensorFlow がロードに失敗する。互換性マトリクスを照会し、推奨バージョン組み合わせを示してください。」

### ハナちゃんサイドノート
> - 切り戻しは「ドライバ → CUDA → cuDNN → フレームワーク」の逆順で行うと事故が少ない。  
> - 将来 CUDA 13 系に移行する際は、まず Docker で動作検証してからホスト刷新を推奨。

---

### 用語集 ― 本書で使用する主要イディオム

| 用語・表記 | 意味・補足 |
|------------|-----------|
| **ワンライナー (one-liner)** | 1 行で完結するシェルコマンドや短いスクリプト |
| **サイドノート (side note)** | 本文から少し外れた補足や警告を差し挟む枠 |
| **ピン留め / pin 留め** | パッケージを固定し、自動アップグレードを抑止する操作 (`apt-mark hold` など) |
| **portproxy** | Windows の `netsh interface portproxy` 機能。指定ポートを別アドレスへ転送する |
| **リモートコンテナモード** | VS Code が WSL や Docker 内で動作し、ホストとは分離された開発環境を提供する機能 |
| **vGPU パススルー** | WSL2 が Windows 側 GPU ドライバを仮想的に利用し、Linux アプリに GPU を共有する仕組み |
| **Compute Capability** | GPU アーキテクチャ世代を表す NVIDIA 指標 (例: 7.5) |
| **ABI 変更** | バイナリ互換性 (Application Binary Interface) が変わること。ライブラリ更新時に要注意 |
| **winget** | Windows 公式の CLI パッケージマネージャ |
| **ldconfig** | Linux で共有ライブラリキャッシュとリンクを再生成するコマンド |
| **nvidia-smi dmon** | `nvidia-smi` のリアルタイム監視サブコマンド (電力・温度・クロック等) |
| **torch.backends.cudnn.benchmark** | PyTorch のカーネル自動最適化フラグ |
| **--n-gpu-layers** | 量子化 LLM 実行時に、モデル層を GPU に割り当てる ggml 系オプション |
| **Docker ランタイムイメージ** | GPU ドライバ入りのコンテナ (`nvidia/cuda:` 系) |
| **wsl --shutdown** | すべての WSL 仮想マシンを停止し、キャッシュをクリアするコマンド |

---


## 12. 奥付

- **タイトル**: 次世代開発環境に捧ぐ福音  
- **副題**: CUDA マシンセットアップ完全攻略ガイド — WW×Si 二人三脚 Edition  
- **著者**: ハナちゃん（Si）／ けんねる（WW）  
- **発行**: 犬小屋研究所技報, 2025 年 6 月  
- **連絡先**: <https://github.com/kennel-org>  
- **免責**: 本書の手順により生じた損害について、著者および発行者は一切責任を負わない。 
