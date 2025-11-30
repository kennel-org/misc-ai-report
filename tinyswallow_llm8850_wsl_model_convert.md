# TinySwallow-1.5B を AX650 (LLM8850) 向けに変換して Raspberry Pi 5 で動かすまで  
WSL + Docker + pulsar2 4.2 メモ

## 0. 免責事項・ライセンス・前提について

1. **非公式ドキュメントです**  
   本記事の内容は、筆者の検証環境での手順メモです。  
   AXERA / SakanaAI / Hugging Face など、各プロジェクトやベンダーの **公式な手順・見解・サポートではありません。**

2. **ライセンスは各配布元の規約に従ってください**  
   - モデル: `SakanaAI/TinySwallow-1.5B-Instruct`  
   - ツール / SDK: AXERA の pulsar2 / AXCL ランタイム・ドライバなど  
   それぞれの **利用条件・商用利用可否・再配布範囲** については、必ず配布元のライセンス・利用規約を確認してください。  
   本記事は「こういう手順で自分の環境で変換できた」という情報共有のみを目的としています。

3. **バージョンや環境はあくまで一例です**  
   - ホスト: WSL2 上の Ubuntu 22.04 (x86_64)  
   - Docker: 29.1.1 (Engine - Community)  
   - pulsar2: 4.2 (AXERA 配布の Docker イメージ `pulsar2:20250924-temp-40ca6580` にタグを付けて利用)  
   - hf CLI: 1.1.6 (`hf` コマンド)  
   - LLM アクセラレータ: AXERA AX650 (LLM8850 ボード)  
   - 推論先: Raspberry Pi 5 (ホスト名 `sirius`)  
   将来のバージョンでは、コマンドやオプション名が変わる可能性があります。適宜読み替えてください。

4. **ホスト名について**  
   - この記事で出てくる `sirius` は、AX650 (LLM8850) ボードを搭載した **Raspberry Pi 5 のホスト名** です。
   - Raspberry Pi 側のユーザー名は `pi` を前提にしていますが、必要に応じて読み替えてください。

---

## 1. ゴールと全体像

### 1-1. ゴール

1. WSL2 上の Ubuntu + Docker で AXERA の pulsar2 (4.2) イメージを動かす。
2. Hugging Face の `SakanaAI/TinySwallow-1.5B-Instruct` をダウンロードし、pulsar2 の `llm_build` で AX650 向けの `*.axmodel` 群を生成する。
3. 生成した `*.axmodel` と埋め込み行列 `model.embed_tokens.weight.bfloat16.bin` を Raspberry Pi 5 (`sirius`) に転送する。
4. 既存の Qwen3 サンプルのランタイム (`main_axcl_aarch64`) をベースに、TinySwallow 用の実行バイナリ `main_axcl_tinyswallow` とランスクリプトを作成する。
5. Raspberry Pi 5 上で TinySwallow-1.5B を LLM8850 (AX650) ボードで動かし、日本語で対話できることを確認する。

### 1-2. 全体構成

- **変換 (ビルド) フェーズ**  
  - 実施場所: WSL2 上の Ubuntu (x86_64)  
  - 役割: pulsar2 (4.2) の Docker コンテナを使って、Hugging Face モデル → `*.axmodel` に変換

- **推論フェーズ**  
  - 実施場所: Raspberry Pi 5 (`sirius`)  
  - 役割: 生成された `*.axmodel` と埋め込みバイナリを読み込み、AX650 で推論を実行

---

## 2. WSL2 側の準備 (Docker + 作業ディレクトリ)

### 2-1. Docker の動作確認

WSL2 上で Docker が動いていることを確認します。

```bash
docker version
docker run hello-world
```

問題なく `Hello from Docker!` が出れば OK です。

### 2-2. Docker を systemd で自動起動にする (任意)

`/etc/wsl.conf` が以下のようになっていることを確認します。

```ini
[boot]
systemd=true
```

Docker デーモンの状態を確認し、有効化します。

```bash
sudo systemctl status docker
sudo systemctl enable --now docker
```

### 2-3. 作業ディレクトリ `~/projects/llm8850` を作成

以下のような構成を前提にします。

```bash
mkdir -p ~/projects/llm8850/{build,models,logs,pulsar2,scripts}
cd ~/projects/llm8850
```

ここに

- `pulsar2` …… AXERA から取得した tar.gz を置いて展開
- `models` …… Hugging Face から落とした TinySwallow のモデルを置く
- `build` …… pulsar2 の `llm_build` の生成物 (`*.axmodel`)
- `logs` …… pulsar2 のビルドログ
- `scripts` …… 補助スクリプト (ビルド用など)

をまとめて配置します。

---

## 3. pulsar2 4.2 Docker イメージの導入

### 3-1. AXERA 公式から pulsar2 イメージを取得

AXERA 公式ドキュメント (pulsar2-docs) から、対象バージョン (ここでは 4.2) 向けの Docker イメージ tar.gz を取得します。  
※実際の URL はドキュメントに記載されているものを使用してください。

```bash
cd ~/projects/llm8850/pulsar2

# 例: AXERA 提供の URL からダウンロード
wget <AXERA が配布している pulsar2 イメージの URL> -O ax_pulsar2_20250924_temp_40ca6580.tar.gz

# Docker イメージとしてロード
sudo docker load -i ax_pulsar2_20250924_temp_40ca6580.tar.gz
```

ロード後、タグを付けてわかりやすくしておきます。

```bash
docker images | grep pulsar2

# 例:
# pulsar2   20250924-temp-40ca6580   b7d809d5c5c0   ...

docker tag pulsar2:20250924-temp-40ca6580 pulsar2:4.2

docker images | grep pulsar2
# pulsar2   4.2                      b7d809d5c5c0   ...
# pulsar2   20250924-temp-40ca6580   b7d809d5c5c0   ...
```

以降は `pulsar2:4.2` という名前で使います。

---

## 4. TinySwallow モデルの取得 (hf CLI)

### 4-1. hf CLI のインストール

Hugging Face Hub の公式 CLI (`hf`) を使います。  
公式のインストールスクリプトは以下。

```bash
curl -LsSf https://hf.co/cli/install.sh | bash
```

> セキュリティポリシーによっては、`curl | bash` を実行する前にスクリプトの中身を確認することを推奨します。

インストール後、`hf` が見えるか確認します。

```bash
which hf
hf version
```

### 4-2. TinySwallow-1.5B-Instruct のダウンロード

作業ディレクトリ配下 `models` に落とします。

```bash
cd ~/projects/llm8850/models
rm -rf TinySwallow-1.5B-Instruct

hf download \
  SakanaAI/TinySwallow-1.5B-Instruct \
  --local-dir TinySwallow-1.5B-Instruct
```

ダウンロード後、構成は以下のようになります。

```bash
cd ~/projects/llm8850/models/TinySwallow-1.5B-Instruct
ls

# 例:
# added_tokens.json   generation_config.json  model.safetensors  special_tokens_map.json  vocab.json
# config.json         LICENSE                 Notice             tokenizer_config.json
# GEMMA_TERMS_OF_USE  merges.txt              README.md          tokenizer.json
```

`config.json` には `model_type: "qwen2"` などが書かれています。  
pulsar2 の `llm_build` はこの設定を見て Qwen2 系として扱うため、出力ファイル名が `qwen2_*.axmodel` になります。

---

## 5. pulsar2 llm_build で AX650 用 .axmodel を生成

### 5-1. pulsar2 コンテナにマウントして起動

WSL 側の `~/projects/llm8850` を `/data` としてコンテナにマウントします。

```bash
cd ~/projects/llm8850

docker run -it --net host --rm \
  -v "$PWD":/data \
  pulsar2:4.2
```

コンテナ内では `/data` にホストの `llm8850` が見えます。

```bash
root@<container>:/# cd /data
ls
# build  logs  models  pulsar2  scripts
```

### 5-2. llm_build の実行

コンテナ内で以下を実行します。

```bash
cd /data
mkdir -p build/TinySwallow-1.5B-Instruct-ax650
mkdir -p logs

pulsar2 llm_build \
  --input_path models/TinySwallow-1.5B-Instruct \
  --output_path build/TinySwallow-1.5B-Instruct-ax650 \
  --hidden_state_type bf16 \
  --prefill_len 128 \
  --kv_cache_len 512 \
  --last_kv_cache_len 512 \
  --chip AX650 \
  --parallel 1 \
  2>&1 | tee logs/tinyswallow_llm_build.log
```

うまくいくと、ログの最後の方に

```text
2025-11-30 16:24:25.105 | SUCCESS  | yamain.command.llm_build:llm_build:214 - prepare llm model done!
...
2025-11-30 17:46:31.057 | SUCCESS  | yamain.command.llm_build:llm_build:311 - build llm model done!
2025-11-30 17:49:02.259 | SUCCESS  | yamain.command.llm_build:llm_build:515 - check llm model done!
```

のようなメッセージが出ます。

### 5-3. 生成物の確認

コンテナを抜けたあと (または別ターミナルから)、WSL 側で確認します。

```bash
cd ~/projects/llm8850
ls build/TinySwallow-1.5B-Instruct-ax650
```

例:

```text
qwen2_p128_l0_together.axmodel   qwen2_p128_l19_together.axmodel  qwen2_p128_l2_together.axmodel
qwen2_p128_l10_together.axmodel  qwen2_p128_l1_together.axmodel   qwen2_p128_l3_together.axmodel
...
qwen2_p128_l27_together.axmodel  qwen2_post.axmodel
```

`config.json` の `model_type: "qwen2"` に合わせて、ファイル名は `qwen2_...` になっているだけで、中身は TinySwallow 用のレイヤです。

---

## 6. 埋め込み行列 (embed) の BF16 バイナリを作成

AXCL 側のランタイムは、トークナイザとは別に埋め込み行列 `model.embed_tokens.weight.bfloat16.bin` を読む前提になっているため、  
`model.safetensors` から埋め込み部分を抜き出して BF16 生バイナリに書き出します。

### 6-1. 仮想環境の作成 (WSL 側)

```bash
cd ~/projects/llm8850/models/TinySwallow-1.5B-Instruct

python3 -m venv venv_tinyswallow_embed
source venv_tinyswallow_embed/bin/activate

pip install --upgrade pip
pip install "torch>=2.2" "safetensors" "packaging"
```

### 6-2. 埋め込みエクスポート用スクリプト

`export_tinyswallow_embed_bf16.py` を作成します。

```bash
cd ~/projects/llm8850/models/TinySwallow-1.5B-Instruct

cat > export_tinyswallow_embed_bf16.py << 'EOF'
import torch
from safetensors.torch import load_file

state = load_file("model.safetensors")
print("Available keys (first 10):", list(state.keys())[:10])

embed = state["model.embed_tokens.weight"]
print("Embedding shape:", embed.shape, "dtype:", embed.dtype)

embed_bf16 = embed.to(torch.bfloat16).contiguous()
embed_bf16.numpy().tofile("model.embed_tokens.weight.bfloat16.bin")

print("Written BF16 embedding to: model.embed_tokens.weight.bfloat16.bin")
EOF
```

実行します。

```bash
python export_tinyswallow_embed_bf16.py
```

成功すると、以下のような出力とファイルができます。

```text
Available keys (first 10): ['model.embed_tokens.weight', ...]
Embedding shape: (151936, 1536), dtype: torch.bfloat16
Written BF16 embedding to: model.embed_tokens.weight.bfloat16.bin
```

```bash
ls -lh model.embed_tokens.weight.bfloat16.bin
# -rw-r--r-- 1 USER USER 446M ... model.embed_tokens.weight.bfloat16.bin
```

このファイルを、後で Raspberry Pi 側の `tinyswallow-1.5b-ax650` ディレクトリにコピーします。

---

## 7. Raspberry Pi 5 (sirius) 側へのファイル転送

### 7-1. .axmodel 一式の転送

WSL 側から `sirius` 上の `~/TinySwallow-1.5B/tinyswallow-1.5b-ax650` にコピーします。

```bash
# WSL 側
cd ~/projects/llm8850

ssh pi@sirius "mkdir -p ~/TinySwallow-1.5B/tinyswallow-1.5b-ax650"

scp build/TinySwallow-1.5B-Instruct-ax650/*.axmodel \
    pi@sirius:~/TinySwallow-1.5B/tinyswallow-1.5b-ax650/
```

### 7-2. 埋め込みバイナリの転送

```bash
cd ~/projects/llm8850/models/TinySwallow-1.5B-Instruct

scp model.embed_tokens.weight.bfloat16.bin \
    pi@sirius:~/TinySwallow-1.5B/tinyswallow-1.5b-ax650/
```

Raspberry Pi 側で確認します。

```bash
ssh pi@sirius
cd ~/TinySwallow-1.5B/tinyswallow-1.5b-ax650
ls -lh
# qwen2_p128_l0_together.axmodel  ...  qwen2_post.axmodel  model.embed_tokens.weight.bfloat16.bin
```

---

## 8. Raspberry Pi 側でのランタイム & Tokenizer 準備

Raspberry Pi 側には、すでに Qwen3 サンプルが動作している構成を前提とします。

### 8-1. 既存 Qwen3 サンプル (参考)

例として、ホームディレクトリ直下に以下のような構成があるとします。

```bash
cd ~
ls -a
# ...
# Qwen3-0.6B/
# ...

cd Qwen3-0.6B
ls -a
# .  ..  main_ax650  main_axcl_aarch64  ...  qwen3-0.6b-ax650  run_qwen3_0.6b_int8_ctx_ax650.sh ...
```

ここから

- `main_axcl_aarch64` …… AXCL ランタイムを使う実行バイナリ
- `qwen3_tokenizer_uid.py` …… Tokenizer HTTP サーバ
- `post_config.json` …… (あれば) ポストプロセス設定

などを TinySwallow 用ディレクトリと共用します。

### 8-2. TinySwallow 用ディレクトリ構成

Raspberry Pi 側で以下のように配置します。

```bash
cd ~
mkdir -p TinySwallow-1.5B/tinyswallow-1.5b-ax650

# 既に scp で *.axmodel と model.embed_tokens.weight.bfloat16.bin はコピー済みの想定

# Qwen3 サンプルからランタイム関連をコピー
cp Qwen3-0.6B/main_axcl_aarch64 TinySwallow-1.5B/
cp Qwen3-0.6B/qwen3_tokenizer_uid.py TinySwallow-1.5B/
cp Qwen3-0.6B/post_config.json TinySwallow-1.5B/  # あれば
```

最終的に、`TinySwallow-1.5B` は以下のような感じになります。

```text
TinySwallow-1.5B/
  ├ main_axcl_aarch64
  ├ qwen3_tokenizer_uid.py
  ├ post_config.json        # あれば
  ├ run_tinyswallow_1.5b_ax650.sh
  └ tinyswallow-1.5b-ax650/
       ├ qwen2_p128_l0_together.axmodel
       ├ ...
       ├ qwen2_p128_l27_together.axmodel
       ├ qwen2_post.axmodel
       └ model.embed_tokens.weight.bfloat16.bin
```

---

## 9. TinySwallow 実行スクリプトの作成

### 9-1. ランタイムバイナリのラッパー `main_axcl_tinyswallow`

既存の `main_axcl_aarch64` を TinySwallow 用にコピーしたうえで、  
AXCL 用のオプションをスクリプト側で指定する運用にします。

```bash
cd ~/TinySwallow-1.5B

cp main_axcl_aarch64 main_axcl_tinyswallow
chmod +x main_axcl_tinyswallow
```

### 9-2. 実行スクリプト `run_tinyswallow_1.5b_ax650.sh`

```bash
cd ~/TinySwallow-1.5B

cat > run_tinyswallow_1.5b_ax650.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

AXMODEL_DIR="$HOME/TinySwallow-1.5B/tinyswallow-1.5b-ax650"
TOKENIZER_URL="http://127.0.0.1:12345"

RUNTIME_BIN="$HOME/TinySwallow-1.5B/main_axcl_tinyswallow"

echo "=== TinySwallow-1.5B on AX650 (llm8850) ==="
echo "  axmodel dir    : ${AXMODEL_DIR}"
echo "  tokenizer URL  : ${TOKENIZER_URL}"
echo "  using runtime  : ${RUNTIME_BIN}"
echo

"${RUNTIME_BIN}" \
  --devices="0" \
  --system_prompt="You are TinySwallow, a helpful Japanese assistant. Reply in Japanese by default." \
  --kvcache_path="" \
  --template_filename_axmodel="${AXMODEL_DIR}/qwen2_p128_l%d_together.axmodel" \
  --filename_post_axmodel="${AXMODEL_DIR}/qwen2_post.axmodel" \
  --url_tokenizer_model="${TOKENIZER_URL}" \
  --filename_tokens_embed="${AXMODEL_DIR}/model.embed_tokens.weight.bfloat16.bin" \
  --axmodel_num=28 \
  --tokens_embed_num=151936 \
  --tokens_embed_size=1536 \
  --use_mmap_load_embed=1 \
  --devices_id="0" \
  --live_print=1
EOF

chmod +x run_tinyswallow_1.5b_ax650.sh
```

> 注意: `--tokens_embed_num` や `--tokens_embed_size` は TinySwallow の `config.json` に合わせています。  
> `config.json` から `vocab_size` と `hidden_size` を確認して設定してください。

---

## 10. Tokenizer サーバの起動と TinySwallow の実行

### 10-1. Tokenizer サーバの起動

Raspberry Pi 側で、別ターミナル (または tmux / screen) を開きます。

```bash
cd ~/TinySwallow-1.5B

# 必要な Python パッケージを仮想環境などでインストールしておく
# 例: transformers, sentencepiece など (Qwen3 サンプルと同様)

python3 qwen3_tokenizer_uid.py
```

実行に成功すると、ログに

```text
Server running at http://0.0.0.0:12345
```

のような表示が出ます。

### 10-2. TinySwallow の起動

別ターミナルで TinySwallow を起動します。

```bash
cd ~/TinySwallow-1.5B
./run_tinyswallow_1.5b_ax650.sh
```

初回は NPU メモリに全レイヤをロードするため、数十秒〜1分程度かかる場合があります。

正常に初期化されると、最後にプロンプト表示とともに対話が始まります。

```text
[I][                            Init][ 264]: LLM init ok
Type "q" to exit, Ctrl+c to stop current running
prompt >> こんばんは。自己紹介してくださいな。
```

入力例:

```text
prompt >> こんばんは。自己紹介してくださいな。
```

応答例:

```text
こんばんは。私はTinySwallowと申します。日本語でサポートするAIアシスタントです。何かお手伝いできることはありますか？ 😊
```

`axcl-smi` で見ると、TinySwallow のプロセスが NPU メモリをそれなりに使っていることが確認できます。

```bash
axcl-smi
```

---

## 11. ハマりポイントまとめ

### 11-1. Hugging Face CLI (`hf`) と `huggingface_hub`

- `huggingface_hub` を `pip install` しても、`huggingface-cli` や `hf` が PATH に入らないケースがあります。
- 公式推奨の `curl -LsSf https://hf.co/cli/install.sh | bash` でインストールすると、`~/.local/bin/hf` が作成され、PATH に追加されます。
- WSL の `~/.bashrc` に `~/.local/bin` を PATH 追加しておくと安心です。

### 11-2. `llm_build` のエラー (HeaderTooLarge)

TinySwallow の `model.safetensors` ダウンロードが途中で壊れていた場合、

```text
safetensors_rust.SafetensorError: Error while deserializing header: HeaderTooLarge
```

のようなエラーが出ることがあります。

- 一度 `models/TinySwallow-1.5B-Instruct` を削除して再ダウンロードすると解消しました。

### 11-3. 埋め込み抽出時の dtype エラー

`safetensors.numpy.load_file` から直接 bfloat16 に変換しようとして失敗したため、

- `safetensors.torch.load_file` + PyTorch のテンソルとして読み込み
- `.to(torch.bfloat16)` で変換

という流れに変更しました。

### 11-4. Raspberry Pi 側での AXCL デバイス権限エラー

`axcl-smi` 実行時や LLM 実行時に

```text
request ports from device 1 fail, errno: 1 Operation not permitted
```

のようなエラーが出る場合があります。

- 古いプロセスが AXCL デバイスを掴んだままになっているケースがありました。
- `ps aux | grep main_axcl` で該当プロセスを探し、`kill` / `kill -9` で終了させると解消しました。
- `axcl-smi` は通常ユーザー (`pi`) で実行しても情報が見える構成になっていれば、sudo は不要です。

### 11-5. プロンプト付きのコマンド例

コマンドをそのままコピペできるように、記事中のシェル例では `pi@sirius:~ $` などのプロンプトは付けていません。  
実行時には、自分の環境のユーザー名・ホスト名に読み替えてください。

---

## 12. おわりに / 今後の展望

- 本記事では、WSL2 上の Ubuntu + Docker + pulsar2 を使って、TinySwallow-1.5B を AX650 (LLM8850) 向け `*.axmodel` に変換し、Raspberry Pi 5 (`sirius`) 上で動かすところまでをまとめました。
- ランタイム自体は Qwen3 用サンプル (`main_axcl_aarch64`) をそのまま流用し、オプションを TinySwallow 用に調整するだけで動作しました。
- 今後のネタとしては、
  1. OpenAI 互換 API 風のサーバを挟んで、他のツールから TinySwallow-1.5B を簡単に呼び出せるようにする。
  2. 複数モデル (Qwen3 / TinySwallow 他) を同一 LLM8850 ボード上で切り替えて使う仕組みを整える。
  3. 温度・トップPなどデコードパラメータを対話的に変更できる UI / TUI を用意する。

あたりが考えられます。  
LLM8850 を載せた Raspberry Pi 5 を「おうちローカル LLM 実験箱」として育てていく第一歩として、何かの参考になればうれしいです。
