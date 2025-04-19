## 要件定義書 — STEP 1: データのダウンロード・正規化・DB登録

### 1. 目的 / Scope
1. **FX 主要通貨ペア**  
   `EURUSD, USDJPY, GBPUSD, AUDUSD, USDCAD, USDCHF, NZDUSD, EURJPY` の **1 分足 (M1)** データを収集する。  
2. 生データを **UTC 基準で正規化**し、欠損・重複を排除したクリーンデータを生成する。  
3. 正規化済みデータを **MySQL 8.x** の `candle` テーブルへ高速ロードする。

### 2. 前提・制約
1. **対象期間は、各通貨ペアで取得可能な最古の M1 データ**から最新（ジョブ実行時点）まで。  
   * 参考値 (2025‑04 時点):  
     - HistData 無償 M1: **2000‑01 〜 2022‑06**  
     - Dukascopy Tick→M1: **2003‑05 〜 現在**  
     - TrueFX Tick: **2009‑01 〜 現在**  
   * スクリプトは API/ディレクトリを走査し **first‑timestamp を自動検出**して期間リストを生成すること。
2. データソース:  
   1) **Dukascopy HTTP ZIP** (一次)  
   2) **TrueFX FTP** (冗長)  
   3) **HistData.com** (補完)  
3. 実行環境: Ubuntu 22.04, Python 3.11, pandas ≥ 2.2, SQLAlchemy ≥ 2.0。  
4. ネットワーク帯域: 下り 50 Mbps 以上、ディスク空き 200 GB 以上。  

### 3. 機能要件

#### 3.1 ディレクトリレイアウト
```
fx_etl/
 ├─ raw/{provider}/{symbol}/{YYYY}/{MM}/  *.zip
 ├─ staging/    # 正規化後 CSV
 ├─ archive/    # 処理済み
 └─ logs/
```

#### 3.2 データベース
* 既定スキーマ: `currency`, `symbol`, `timeframe`, `candle`。  
* 主キー: **(symbol_id, tf, ts_utc)** — `provider_id` は重複判定に含めない。  
* パーティション: RANGE 年次 + HASH(symbol_id)。

#### 3.3 データソース別正規化要件

| ソース | 入力形式 | タイムゾーン | Volume | 正規化ステップ |
|-------|---------|-------------|--------|---------------|
| **Dukascopy** | `BID_ASK.csv` (ZIP) | UTC/EET (要 UTC) | Tick 数 | ① UTC モードで取得 ② mid 価格生成 ③ 重複バー最新採用 |
| **HistData** | CSV (ZIP) | GMT | ロット数 | ① col rename ② UTC 直置き ③ 欠損は NaN |
| **TrueFX** | Tick CSV | GMT | 契約数 | ① Tick→M1 集約 ② 異常値除去 ③ UTC |

#### 3.4 重複排除ポリシー
* 同一バー複数プロバイダ→ **優先順位**: 1) Dukascopy 2) HistData 3) TrueFX 4) その他  
* 落選バーは `dup_discard_log` に保存。  
* プロバイダ間乖離 ±2 pips 超は `provider_divergence` に格納。

#### 3.5 欠損 M1 データの機械生成
1. 欠損判定: ペア×日でレコード数≠1440。  
2. 生成モデル: Gradient Boosting Regressor (特徴: 前後 60 分の価格・ボラ、ブローカー種別、曜日・時刻)。  
3. 品質検証: MAPE < 0.5 ‰ & スプレッド許容範囲内。  
4. ダミーフラグ `is_synthetic` を立てて `candle` へ INSERT。

---

## 4. 改善オプション（参考）

### 4.1 オーケストレーション & DevOps
1. DAG 管理 (Airflow/Dagster)  
2. Docker & CI/CD (GitHub Actions)  
3. Prometheus + Grafana 監視

### 4.2 品質保証
4. Great Expectations / pydantic による値域検証  
5. データ指紋 (SHA‑256) を `dataset_snapshot` に保存

### 4.3 パフォーマンス & 拡張性
6. `aiohttp` 非同期ダウンロード (50 並列制限)  
7. 年次 RANGE + HASH(symbol) パーティション  
8. Parquet + DuckDB/ClickHouse ミラー

### 4.4 セキュリティ & コスト
9. Secrets (.env / Docker Secret / Vault)  
10. Raw ZIP を 30 日後に S3 Glacier Deep Archive

### 4.5 ドキュメント & テスト
11. サンプルデータで統合テスト (GitHub Actions)  
12. README にローカル実行手順 & スキーママイグレーション方法を記載
