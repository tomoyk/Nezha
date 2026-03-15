# Nezha リポジトリ調査・要件分析

日付: 2026-03-15

## 概要
Nezha（FSE 2023論文実装）のセットアップ要件・入力データ仕様を調査し，Rook Ceph OSD 障害シナリオへの適用可否を検討した．

## 作業内容

### 1. Nezha の概要・起動要件調査

- リポジトリ全体のディレクトリ構造・README・INSTALL.md を調査
- 必要なソフトウェア・依存関係を整理

**必要なソフトウェア：**

| ソフトウェア | バージョン |
|---|---|
| Python | 3.6 推奨 |
| drain3 | 0.9.10 |
| pandas | 0.23.4 |
| numpy | 1.15.4 |
| matplotlib | 3.3.4 |
| PyYAML | 6.0.1 |
| psutil | 5.9.0 |
| more_itertools | 8.12.0 |

**実行コマンド：**

```bash
python3.6 -m pip install -r requirements.txt

# OnlineBoutique サービスレベル
python3.6 ./main.py --ns hipster --level service

# TrainTicket 内部サービスレベル
python3.6 ./main.py --ns ts --level inner
```

**Prometheus 連携：** コードベース内に Prometheus 連携機能なし．メトリクスは CSV ファイルから読み込む設計．

### 2. 入力データ仕様の調査

実際の CSV ファイルを読み込み，各入力ファイルのカラム構造を確認．

**必要な入力ファイル（時間帯ごと）：**

| ファイル | 主要カラム |
|---|---|
| `log/HH_MM_log.csv` | TimeUnixNano, PodName, TraceID, SpanID, Log(JSON) |
| `metric/{PodName}_metric.csv` | TimeStamp, PodName, CpuUsageRate(%), MemoryUsageRate(%), NetworkP90(ms) ほか |
| `trace/HH_MM_trace.csv` | TraceID, SpanID, ParentID, PodName, StartTimeUnixNano, EndTimeUnixNano |
| `traceid/HH_MM_traceid.csv` | TraceID のみ（ヘッダーなし） |
| `metric/dependency.csv` | Source, Target, Weight |
| `metric/front_service.csv` | ServiceName, SuccessRate(%), LatencyP90(s) ほか |

**JSON ラベルファイル：**

| ファイル | 用途 |
|---|---|
| `rca_data/YYYY-MM-DD/YYYY-MM-DD-fault_list.json` | 障害注入情報（時刻・ポッド・障害種別） |
| `construct_data/root_cause_hipster.json` | 内部サービスレベル正解ラベル |

**設定ファイル：**

- `metric_threshold/{service}.csv`：正常時の平均値・標準偏差（2行）
- `log_template/drain3_hipster.ini`：Drain3 設定（類似度閾値・マスキングルール）
- `log_template/hipster.bin`：学習済み Drain3 モデル

**データ量目安：** ログ約25,000行・メトリクス約1,200行・トレース約23,000行 /時間帯

### 3. Rook Ceph OSD 障害シナリオへの適用可否分析

**シナリオ：** OSD Pod クラッシュ → PV マウント失敗 → Application Pod エラー

**結論：原因特定は困難**

| 理由 | 詳細 |
|---|---|
| 分散トレースに OSD が現れない | Nezha のイベントグラフは TraceID/SpanID でリンクされたスパンから構築される．OSD はトレース非参加のため因果グラフに入らない |
| 障害モードがリクエスト失敗ではない | PV マウント失敗は Pod 起動失敗であり，スパンとして記録されない |
| dependency.csv にストレージ依存がない | HTTP/gRPC 呼び出し関係のみ対象．App→PV→OSD の依存は表現できない |
| メトリクス監視対象外 | OSD の I/O 異常・ディスク障害は追跡しない |

**Nezha が部分的に検出できること：**
- Application Pod のログ・メトリクス異常は検出可能
- ただし「OSD クラッシュが根本原因」という診断は不可

**対応に必要な拡張：**
1. `dependency.csv` に `App → PVC → OSD` のストレージ依存追加
2. OSD Pod のメトリクス収集・閾値設定
3. Kubernetes イベント（マウントタイムアウト等）をログとして統合
4. インフラ障害の波及モデルの実装（現行はリクエスト伝播モデルのみ）

## 変更ファイル一覧

なし（調査・分析のみ，コード変更なし）

## 結果

- Nezha はマイクロサービス間の HTTP/gRPC 呼び出しチェーン障害の RCA に特化したツール
- Kubernetes ストレージインフラ起因の障害はスコープ外
- Prometheus データを使う場合は `/api/v1/query_range` から取得して CSV 変換するアダプタが必要
- Rook Ceph OSD 障害シナリオには，Kubernetes イベント・ストレージメトリクスを統合した別の RCA アプローチが適切

## 備考

- 対象データセット：OnlineBoutique（hipster）と TrainTicket（ts）の2種類
- 評価指標：AS@1（Top-1精度）で hipster 92.9%，ts 86.7%
- 障害種別：`cpu_consumed`, `cpu_contention`, `network_delay`, `return`, `exception` の5種類
