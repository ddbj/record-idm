# record-idm

DDBJ Record ID Manager: DDBJ が管理する accession の status・relation・登録者情報を統一的に管理するシステム。

> **現在のフェーズ: 仕様策定中。** このリポジトリでは `docs/` 以下で仕様の設計・議論を行っている。実装は仕様が固まった後に着手する予定。

## 背景

### DDBJ のリポジトリ群

DDBJ は以下のリポジトリを管理している。それぞれが独自の管理 DB・ファイル形式を持つ。

| リポジトリ       | 管理方式                                   | 登録口           |
| ---------------- | ------------------------------------------ | ---------------- |
| Trad（塩基配列） | PostgreSQL（g-actual, w-actual, a-actual） | D-way / MSS      |
| BioProject       | PostgreSQL                                 | D-way            |
| BioSample        | PostgreSQL                                 | D-way            |
| DRA（SRA 自極）  | D-way 内部 + Accessions.tab                | D-way            |
| GEA              | DB + IDF/SDRF ファイル                     | D-way            |
| JGA / AGD        | 申請管理システム + XML/CSV                 | 申請管理システム |
| MetaboBank       | IDF/SDRF + status ファイル                 | 手作業           |
| JVar             | Excel メタデータ                           | 手作業           |

INSDC データ交換により、管理 DB にないデータ（NCBI・EBI 由来の SRA accession 等）もファイルベースで存在する。

### 現状の課題

1. **status 定義がリポジトリごとにバラバラ**
   - Trad は `public / private / killed`、BioProject は `5100 / 5200 / ... / 5800`、SRA は `live / unpublished / suppressed / withdrawn` と、status の enum がリポジトリごとに異なる
   - 公開状態と投稿処理段階（審査中・差し戻し等）が混在した status 値も多い（例: BioProject の `submitted(5100)` / `curating(5200)` / `private(5400)` は全て「未公開」だが投稿処理段階が異なる）

2. **統一的な status list が存在しない**
   - livelist のフォーマットがリポジトリごとに異なり、出力しているものと出力していないものがある
   - 既存の livelist は INSDC 提出用であり、DDBJ 内部向けに全リポジトリを横断する統一フォーマットがない

3. **relation が散在している**
   - accession 間の関係（BioProject ↔ BioSample、BioProject ↔ SRA 等）が複数の DB・ファイルに分散しており、横断的に辿る仕組みがない
   - 関連データの連動公開（SRA 公開時に対応する BioProject・BioSample も公開する等）を最新 status で動的に行えない

4. **リアルタイム性の欠如**
   - 現在は日次バッチで status を処理しており、データ量が一日で捌けない場合に公開遅延が発生しうる
   - 元ソース DB の変更を検知して反映する仕組みがない

### 2020 年の DDBJ-LD 仕様からの進化

2020 年に策定された「DDBJ-LD：レコードステイタス仕様書」では、レコードのライフサイクル全体を 1 つの status 値（unsubmitted / submitted / private / public / suppressed / replaced / killed / cancel）と、`visibility` / `searchability` の boolean フラグで管理する設計だった。

record-idm はこれを再設計し、以下の変更を加えている:

- **status を 2 次元に分離**: データの公開状態（`record_status`）と投稿処理段階（`submission_stage`）を独立した軸として管理する。これにより INSDC 標準との互換性を維持しつつ、リポジトリごとの粒度の違いを吸収できる
- **`visibility` / `searchability` を廃止**: `record_status` から一意に導出できるため、独立プロパティとしては持たない
- **`replaced` を status から relation へ移動**: Trad の `secondary` や SRA の `replaced` は accession 間の関係性であり、公開状態ではない

## record-idm が管理するもの

record-idm は 3 つの管理対象を持つ。

### 1. Status（accession の状態）

accession ごとに `record_status` × `submission_stage` の 2 次元で状態を管理する。

| 次元               | 意味                       | 値                                                                                |
| ------------------ | -------------------------- | --------------------------------------------------------------------------------- |
| `record_status`    | データの公開状態           | `private`, `public`, `suppressed`, `withdrawn`, `canceled`, `unregistered`        |
| `submission_stage` | 投稿処理の段階（nullable） | `draft`, `submitted`, `in_curation`, `revision_requested`, `accepted`, `rejected` |

`record_status` の前半 4 値（private / public / suppressed / withdrawn）は INSDC 標準と互換。`canceled` / `unregistered` は DDBJ 拡張。

### 2. Relation（accession 間の関係）

BioProject を頂点とする DAG として accession 間の関係を管理する。relation が定義されている accession 同士は、status 変更時に連動する。

### 3. Submitter（登録者情報）

accession と登録者（`ddbj_account_id`）の紐づけを管理する。自極分については D-way の submission に ddbj_account_id が紐づいている。全リポジトリでの実現は将来課題。

## 仕様書・調査ドキュメント

`docs/` が各機能の SSOT（Single Source of Truth）となる。

### 仕様

| ドキュメント                                         | 内容                                                                                                             |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| [docs/data-model.md](docs/data-model.md)             | データモデル全体像。status の 2 次元モデル（`record_status` × `submission_stage`）、各リポジトリからのマッピング |
| [docs/record-status.md](docs/record-status.md)       | `record_status` の定義値、INSDC 標準との対応、状態遷移                                                           |
| [docs/submission-stage.md](docs/submission-stage.md) | `submission_stage` の定義値、状態遷移                                                                            |
| [docs/relations.md](docs/relations.md)               | accession 間の relation 定義、リポジトリ間・リポジトリ内の edge 一覧                                             |

### 調査

| ドキュメント                                 | 内容                                                   |
| -------------------------------------------- | ------------------------------------------------------ |
| [docs/repositories.md](docs/repositories.md) | 各リポジトリのデータソース（DB, ファイル）の所在と構造 |

## 関連リポジトリ

- [ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) — DDBJ Search 向けに各リポジトリのデータを JSONL に変換するコンバータ。relation（dblink）の構築、status の parse、実データの path など、record-idm の仕様設計で参照する既存実装が含まれる

## 参考資料

- [DDBJ-LD：レコードステイタス仕様書（2020）](https://ddbj-dev.atlassian.net/wiki/spaces/D/pages/328433665/DDBJ-LD) — 2020 年に策定されたレコードステイタスの初期仕様。record-idm はこれを参考に、`record_status` と `submission_stage` の 2 次元モデルとして再設計している
- [INSDC Status Document](https://www.insdc.org/submitting-standards/insdc-status-document/) — INSDC が定める accession の公開状態の標準仕様

## License

[Apache-2.0](LICENSE)
