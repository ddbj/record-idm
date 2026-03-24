# データモデル

record-idm のデータモデルを定義する。

## Status の 2 次元モデル

record-idm は accession の状態を 2 つの次元で管理する。

| 次元 | 意味 | 例 |
|------|------|------|
| [`record_status`](./record-status.md) | データの公開状態（外の世界から見た到達可能性） | live, suppressed, withdrawn |
| [`submission_stage`](./submission-stage.md) | 投稿処理の段階（DDBJ 内部のワークフロー状態） | submitted, in_curation, accepted |

### なぜ 2 次元か

既存の各リポジトリの status 値には、公開状態と投稿処理段階が混在している。例えば BioProject/BioSample の `submitted(5100)`, `curating(5200)`, `private(5400)` は全て「未公開」だが、投稿処理の段階が異なる。これらを 1 つの値に押し込むと情報が失われる。

2 次元に分離することで:

- `record_status` は INSDC 標準と互換性を維持できる
- `submission_stage` で投稿処理の詳細な段階を追跡できる
- 各リポジトリが持つ粒度の違いを吸収できる（`submission_stage` は nullable）

### 設計判断の経緯

- [DDBJ-LD：レコードステイタス仕様書（2020）](https://ddbj-dev.atlassian.net/wiki/spaces/D/pages/328433665/DDBJ-LD) では、全ライフサイクルを 1 つの値（unsubmitted, submitted, private, public, suppressed, replaced, killed, cancel）にまとめ、`visibility` / `searchability` の boolean フラグで意味を補足する設計だった
- `visibility` / `searchability` は `record_status` から導出可能なため、独立したプロパティとしては持たない
- Schema.org は `creativeWorkStatus`（コンテンツのライフサイクル）と `actionStatus`（プロセスの実行状態）で暗黙的に 2 次元を分離している
- [INSDC Status Document](https://www.insdc.org/submitting-standards/insdc-status-document/) は accession の公開状態（live, unpublished, suppressed, withdrawn）のみ標準化し、submission workflow は各アーカイブの内部実装に委ねている。record-idm はこの暗黙的な分離を明示化した形になる

### record_status × submission_stage の制約

2 つの次元は完全に独立ではなく、以下の制約がある。

| record_status | 許容される submission_stage |
|---|---|
| `unpublished` | `draft`, `submitted`, `in_curation`, `revision_requested`, `accepted` |
| `live` | `accepted` |
| `suppressed` | `accepted` |
| `withdrawn` | `accepted` |
| `canceled` | `draft`, `submitted`, `in_curation`, `rejected` |
| `unregistered` | null |

- `submission_stage` は主に `unpublished` の内訳を詳細化する役割を持つ
- 公開後（`live`, `suppressed`, `withdrawn`）は基本的に `accepted` 固定
- `canceled` は公開前に中止されたレコードであり、`submission_stage` は中止時点の段階を保持する

## 各リポジトリの status マッピング

各リポジトリ固有の status 値から `record_status` × `submission_stage` への変換。データソースの詳細は [repositories.md](./repositories.md) を参照。

### Trad

ソース: `manager` テーブルの `status` カラム

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| private (1001) | `unpublished` | null | hold 期間中 |
| public (1002) | `live` | null | |
| suppressed (1004) | `suppressed` | null | |
| secondary (1005) | `suppressed` | null | relation `replaced_by` も設定する |
| killed (1006) | `withdrawn` | null | |
| unregistered (1007) | `unregistered` | null | |

### BioProject / BioSample

ソース: `mass.project` / `mass.sample` テーブルの `status_id` カラム

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| submitted (5100) | `unpublished` | `submitted` | |
| curating (5200) | `unpublished` | `in_curation` | |
| private (5400) | `unpublished` | `accepted` | accession 発行済み、hold 期間中 |
| public (5500) | `live` | `accepted` | |
| killed (5600) | `withdrawn` | `accepted` | |
| canceled (5700) | `canceled` | null | |
| suppressed (5800) | `suppressed` | `accepted` | |
| 5900 | ? | ? | 要調査 |

### SRA（全極、NCBI 作成）

ソース: SRA_Accessions.tab の `Status` カラム

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| live | `live` | null | |
| unpublished | `unpublished` | null | |
| suppressed | `suppressed` | null | |
| withdrawn | `withdrawn` | null | |
| replaced | `withdrawn` | null | relation `replaced_by` も設定する |

### DRA（自極、DDBJ 作成）

ソース: DRA_Accessions.tab（公開済みのみ出力。未公開分は D-way 内で管理）

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| public | `live` | null | |
| suppressed | `suppressed` | null | |
| withdrawn | `withdrawn` | null | |

### GEA

ソース: livelist ファイル（公開済みのみ）

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| Public | `live` | null | |
| Permanently Suppressed | `suppressed` | null | |
| Withdrawn | `withdrawn` | null | |

### JGA

ソース: 申請管理システムの `appl_status_type`（accession 未発行のものも管理対象）

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| 申請書類作成中 (10) | `unpublished` | `draft` | accession 未発行 |
| 申請完了 (20) | `unpublished` | `submitted` | accession 未発行 |
| 差し戻し中 (30) | `unpublished` | `revision_requested` | accession 未発行 |
| 審査中 (40) | `unpublished` | `in_curation` | accession 未発行 |
| 却下 (50) | `canceled` | `rejected` | accession 未発行 |
| 承認 (60) | `live` | `accepted` | accession 発行済み |
| 取り下げ (70) | `canceled` | null | accession 未発行 or 発行済み。要調査 |
| 利用期間終了 (80) | TODO | TODO | 要調査 |

### AGD

JGA と同じ構成。

### MetaboBank

ソース: study ディレクトリ内の status file

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| Public (Release Date あり) | `live` | `accepted` | |
| Private (Release Date なし) | `unpublished` | `accepted` | |
| In review | `unpublished` | `in_curation` | |
| Cancelled | `canceled` | null | |
| Killed | `withdrawn` | `accepted` | |
| Temporarily suppressed | `suppressed` | `accepted` | |
| Permanently suppressed | `suppressed` | `accepted` | |

### JVar

ソース: Excel メタデータの `Hold/Release` フィールド

| 生 status | → record_status | → submission_stage | 備考 |
|---|---|---|---|
| Release | `live` | null | |
| Hold | `unpublished` | null | |

## Relation

TODO: accession 間の関係性（BioProject を頂点とする DAG、`replaced_by` 等）のモデルを定義する。
