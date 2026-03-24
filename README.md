# record-idm

DDBJ Record ID Manager: DDBJ が管理する accession の status・relation・登録者情報を統一的に管理するシステム。

> **現在のフェーズ: 仕様策定中。** このリポジトリでは `docs/` 以下で仕様の設計・議論を行っている。実装は仕様が固まった後に着手する予定。

## 背景

DDBJ の各リポジトリ（Trad, BioProject, BioSample, DRA, GEA, JGA, AGD, MetaboBank, JVar）は、それぞれ独自の管理 DB・ファイル形式で accession の status を管理している。リポジトリごとに status の表現が異なり（例: Trad は `public/private/killed`、BioProject は `5100/5200/.../5800`、SRA は `live/unpublished/suppressed/withdrawn`）、統一的な問い合わせや横断的な管理が困難になっている。

record-idm はこれらを統一的に扱い、以下を実現する:

- accession の公開状態（`record_status`）と投稿処理段階（`submission_stage`）を共通の定義値で管理する
- accession 間の relation を管理する（BioProject を頂点とする DAG）
- accession と登録者（ddbj_account_id）の紐づけを管理する
- status list の JSONL dump を提供する

## 仕様書・調査ドキュメント

`docs/` が各機能の SSOT（Single Source of Truth）となる。

### 仕様

| ドキュメント | 内容 |
|---|---|
| [docs/data-model.md](docs/data-model.md) | データモデル全体像。status の 2 次元モデル（`record_status` × `submission_stage`）、各リポジトリからのマッピング |
| [docs/record-status.md](docs/record-status.md) | `record_status` の定義値、INSDC 標準との対応、状態遷移 |
| [docs/submission-stage.md](docs/submission-stage.md) | `submission_stage` の定義値、状態遷移 |

### 調査

| ドキュメント | 内容 |
|---|---|
| [docs/repositories.md](docs/repositories.md) | 各リポジトリのデータソース（DB, ファイル）の所在と構造 |

## 参考資料

- [DDBJ-LD：レコードステイタス仕様書（2020）](https://ddbj-dev.atlassian.net/wiki/spaces/D/pages/328433665/DDBJ-LD) — 2020 年に策定されたレコードステイタスの初期仕様。record-idm はこれを参考に、`record_status` と `submission_stage` の 2 次元モデルとして再設計している
- [INSDC Status Document](https://www.insdc.org/submitting-standards/insdc-status-document/) — INSDC が定める accession の公開状態の標準仕様

## License

[Apache-2.0](LICENSE)
