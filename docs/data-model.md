# データモデル

record-idm のデータモデルを定義する。

## Status の 2 次元モデル

record-idm は accession の状態を 2 つの次元で管理する。

| 次元                                        | 意味                                           | 例                               |
| ------------------------------------------- | ---------------------------------------------- | -------------------------------- |
| [`record_status`](./record-status.md)       | データの公開状態（外の世界から見た到達可能性） | live, suppressed, withdrawn      |
| [`submission_stage`](./submission-stage.md) | 投稿処理の段階（DDBJ 内部のワークフロー状態）  | submitted, in_curation, accepted |

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

| record_status  | 許容される submission_stage                                           |
| -------------- | --------------------------------------------------------------------- |
| `unpublished`  | `draft`, `submitted`, `in_curation`, `revision_requested`, `accepted` |
| `live`         | `accepted`                                                            |
| `suppressed`   | `accepted`                                                            |
| `withdrawn`    | `accepted`                                                            |
| `canceled`     | null, `draft`, `submitted`, `in_curation`, `rejected`                 |
| `unregistered` | null                                                                  |

- `submission_stage` は主に `unpublished` の内訳を詳細化する役割を持つ
- 公開後（`live`, `suppressed`, `withdrawn`）は基本的に `accepted` 固定
- `canceled` は公開前に中止されたレコードであり、`submission_stage` は中止時点の段階を保持する。段階情報を持たないリポジトリ（BioProject/BioSample 等）では null となる

## 各リポジトリの status マッピング

各リポジトリ固有の status 値から `record_status` × `submission_stage` への変換。データソースの詳細は [repositories.md](./repositories.md) を参照。

### Trad

ソース: `manager` テーブルの `status` カラム（record_status）+ `dataflow` テーブル（submission_stage、nullable）

| Raw Status          | → record_status | → submission_stage | 備考                              |
| ------------------- | --------------- | ------------------ | --------------------------------- |
| private (1001)      | `unpublished`   | （※）              | hold 期間中                       |
| public (1002)       | `live`          | `accepted`         |                                   |
| suppressed (1004)   | `suppressed`    | `accepted`         |                                   |
| secondary (1005)    | `suppressed`    | `accepted`         | relation `replaced_by` も設定する |
| killed (1006)       | `withdrawn`     | `accepted`         |                                   |
| unregistered (1007) | `unregistered`  | null               |                                   |

（※）`private` の `submission_stage` は `dataflow` テーブルの最新ステータスから導出する。`dataflow` は accession 単位の細粒度な作業ログであり、全 accession にレコードがあるとは限らないため、取得できない場合は null となる。

| dataflow status group                     | → submission_stage   |
| ----------------------------------------- | -------------------- |
| 1001-1003（received / acknowledged）      | `submitted`          |
| 1007-1011（annotation / review）          | `in_curation`        |
| 1012（sent to submitter）                 | `revision_requested` |
| 1030（return from reviewer）              | `in_curation`        |
| 1033 以降（all done / ready for release） | `accepted`           |
| レコードなし                              | null                 |

### BioProject / BioSample

ソース: `mass.project` / `mass.sample` テーブルの `status_id` カラム

| Raw Status                    | → record_status | → submission_stage | 備考                                                      |
| ----------------------------- | --------------- | ------------------ | --------------------------------------------------------- |
| submitted (5100)              | `unpublished`   | `submitted`        |                                                           |
| curating (5200)               | `unpublished`   | `in_curation`      |                                                           |
| private (5400)                | `unpublished`   | `accepted`         | accession 発行済み、hold 期間中                           |
| public (5500)                 | `live`          | `accepted`         |                                                           |
| killed (5600)                 | `withdrawn`     | `accepted`         |                                                           |
| canceled (5700)               | `canceled`      | null               |                                                           |
| suppressed (5800)             | `suppressed`    | `accepted`         |                                                           |
| temporarily_suppressed (5900) | `suppressed`    | `accepted`         | 一時的な非公開化（登録者依頼による再公開前提の suppress） |

### SRA（全極、NCBI 作成）

ソース: SRA_Accessions.tab の `Status` カラム

| Raw Status  | → record_status | → submission_stage | 備考                              |
| ----------- | --------------- | ------------------ | --------------------------------- |
| live        | `live`          | null               |                                   |
| unpublished | `unpublished`   | null               |                                   |
| suppressed  | `suppressed`    | null               |                                   |
| withdrawn   | `withdrawn`     | null               |                                   |
| replaced    | `suppressed`    | null               | relation `replaced_by` も設定する |

### DRA（自極、DDBJ 作成）

ソース: drmdb `mass.status_history` テーブル（event sourcing、最新 status で判定）

| Raw Status                 | → record_status | → submission_stage   | 備考                     |
| -------------------------- | --------------- | -------------------- | ------------------------ |
| 100 (draft)                | `unpublished`   | `draft`              |                          |
| 190 (canceled)             | `canceled`      | null                 |                          |
| 300 (submitted)            | `unpublished`   | `submitted`          |                          |
| 380 (validated)            | `unpublished`   | `in_curation`        | 自動 validation 完了後   |
| 390 (revision requested)   | `unpublished`   | `revision_requested` |                          |
| 400 (processing)           | `unpublished`   | `in_curation`        | キュレーション処理中     |
| 500 (accessioned)          | `unpublished`   | `accepted`           | accession 発行済み       |
| 700 (held)                 | `unpublished`   | `accepted`           | hold 期間中              |
| 750 (pre-release)          | `unpublished`   | `accepted`           | 公開処理中（中間状態）   |
| 770 (suppressed)           | `suppressed`    | `accepted`           |                          |
| 800 (public)               | `live`          | `accepted`           |                          |
| 1000 (withdrawn)           | `withdrawn`     | `accepted`           |                          |
| 1100 / 1200 (intermediate) | （※）           | `accepted`           | suppress/withdraw 処理中 |

（※）1100/1200 は公開後の管理操作（suppress/withdraw 処理の中間状態）。record_status は遷移元に依存する。

注: DRA_Accessions.tab は公開済みデータのみ出力するファイルであり、`public` / `suppressed` / `withdrawn` の 3 status のみ含む。record-idm は drmdb を正規ソースとし、未公開分を含む全ライフサイクルをカバーする。

### GEA

ソース: dordb `mass.submission_status_history` テーブル（submission 単位のワークフロー状態）

| Raw Status                       | → record_status | → submission_stage   | 備考                     |
| -------------------------------- | --------------- | -------------------- | ------------------------ |
| 0 (created)                      | `unpublished`   | `draft`              |                          |
| 10-60 (submit → validate → load) | `unpublished`   | `submitted`          | 自動処理パイプライン     |
| 100-130 (errors)                 | `unpublished`   | `revision_requested` | 処理エラーによる差し戻し |
| 200 (validated)                  | `unpublished`   | `in_curation`        |                          |
| 210 (curating)                   | `unpublished`   | `in_curation`        |                          |
| 220-240 (accession → data load)  | `unpublished`   | `accepted`           | accession 発行以降       |
| 250 (data error)                 | `unpublished`   | `revision_requested` | データロードエラー       |
| 260 (private)                    | `unpublished`   | `accepted`           | hold 期間中              |
| 270 (wait release)               | `unpublished`   | `accepted`           | 公開待ち                 |
| 300 (public)                     | `live`          | `accepted`           |                          |
| 400 (canceled)                   | `canceled`      | null                 |                          |
| 410 (suppressed)                 | `suppressed`    | `accepted`           |                          |
| 420 (killed)                     | `withdrawn`     | `accepted`           |                          |

注: livelist ファイルは公開済みデータのみ含む（Public / Permanently Suppressed / Withdrawn）。record-idm は dordb を正規ソースとし、未公開分を含む全ライフサイクルをカバーする。

### JGA

ソース: 申請管理システムの `appl_status_type`（accession 未発行のものも管理対象）

| Raw Status                        | → record_status | → submission_stage   | 備考                                                           |
| --------------------------------- | --------------- | -------------------- | -------------------------------------------------------------- |
| 申請書類作成中 (10)               | `unpublished`   | `draft`              | accession 未発行                                               |
| 申請完了 (20)                     | `unpublished`   | `submitted`          | accession 未発行                                               |
| 差し戻し中 (30)                   | `unpublished`   | `revision_requested` | accession 未発行                                               |
| 審査中 (40)                       | `unpublished`   | `in_curation`        | accession 未発行                                               |
| 却下 (50)                         | `canceled`      | `rejected`           | accession 未発行                                               |
| 承認 (60)                         | `live`          | `accepted`           | accession 発行済み                                             |
| 取り下げ (70)、accession 未発行   | `canceled`      | null                 | accession は承認（60）後に付与される前提                       |
| 取り下げ (70)、accession 発行済み | `withdrawn`     | `accepted`           | 承認（60 = `live`）を経由した後の取り下げ                      |
| 利用期間終了 (80)                 | （対応不要）    | （対応不要）         | データ利用申請の status であり、レコード自体の公開状態ではない |

### AGD

JGA と同じ構成。

### MetaboBank

ソース: study ディレクトリ内の status file

| Raw Status                  | → record_status | → submission_stage | 備考 |
| --------------------------- | --------------- | ------------------ | ---- |
| Public (Release Date あり)  | `live`          | `accepted`         |      |
| Private (Release Date なし) | `unpublished`   | `accepted`         |      |
| In review                   | `unpublished`   | `in_curation`      |      |
| Cancelled                   | `canceled`      | null               |      |
| Killed                      | `withdrawn`     | `accepted`         |      |
| Temporarily suppressed      | `suppressed`    | `accepted`         |      |
| Permanently suppressed      | `suppressed`    | `accepted`         |      |

### JVar

ソース: Excel メタデータの `Hold/Release` フィールド

| Raw Status | → record_status | → submission_stage | 備考 |
| ---------- | --------------- | ------------------ | ---- |
| Release    | `live`          | null               |      |
| Hold       | `unpublished`   | null               |      |

## Relation

accession 間の関係性（BioProject を頂点とする DAG、各リポジトリ間・内部の edge 定義）は [relations.md](./relations.md) を参照。
