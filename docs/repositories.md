# リポジトリ一覧とデータソース

DDBJ が管理するリポジトリと、各リポジトリの管理 DB・ファイルの所在をまとめる。

## 一覧

**INSDC データ交換対象リポジトリ**

| リポジトリ | 自極データ                                                           | 他極データ                                        | 登録口           | 備考                        |
| ---------- | -------------------------------------------------------------------- | ------------------------------------------------- | ---------------- | --------------------------- |
| Trad       | PostgreSQL (g-actual, e-actual, w-actual)                            | GenBank flatfile, EMBL flatfile                   | D-way (NSSS/MSS) | ~1.87 億件、塩基配列        |
| BioProject | PostgreSQL (a011:54306), XML, livelist                               | BP XML (3.6 GB)                                   | D-way            | 自極 ~4.2 万                |
| BioSample  | PostgreSQL (a011:54306), XML, livelist                               | BS XML (4.3 GB gz)                                | D-way            | 自極 ~187 万、全極 ~4000 万 |
| SRA (DRA)  | PostgreSQL (drmdb, a011:54306), DRA_Accessions.tab, DRA_Metadata.tar | SRA_Accessions.tab (30 GB), NCBI_SRA_Metadata.tar | D-way            | 全極 ~1.44 億、自極 ~230 万 |

**DDBJ 独自リポジトリ**

| リポジトリ | DB / データソース                                   | 備考                                        |
| ---------- | --------------------------------------------------- | ------------------------------------------- |
| GEA        | PostgreSQL (dordb, a011:54306), IDF/SDRF + livelist | ~682 件                                     |
| JGA        | XML + CSV（申請管理システムから出力）               | ~531 study, ~664 dataset, controlled-access |
| AGD        | PostgreSQL（申請管理システム）                      | JGA とほぼ同じ構成                          |
| MetaboBank | IDF/SDRF                                            | ~156 study、メタボロミクス                  |
| JVar       | ファイルベース                                      | 2 study 公開中、将来 dbSNP/dbVar と交換予定 |

**参照データ**

| リポジトリ            | データソース                            | 備考                                            |
| --------------------- | --------------------------------------- | ----------------------------------------------- |
| NCBI GenBank Assembly | assembly_summary_genbank.txt (NCBI FTP) | insdc-assembly, insdc-master の relation ソース |
| NCBI taxonomy         | tax dump (NCBI FTP)                     | 生物種分類情報                                  |

## INSDC データ交換対象リポジトリ

### Trad

INSDC flatfile（アノテーション付き塩基配列）全体を指す DDBJ 内部呼称。"Traditional" の略で、DRA, GEA 等の新しい DB 群と区別するための名前。NSSS（Web 登録）/ MSS（大量登録）経由で登録される。WGS, TSA, TPA, CON, EST 等の division も含む。

#### 自極データ

PostgreSQL 3 インスタンス `@a012`。テーブル構造は 3 DB 共通。

| DB 名    | ポート | 内容                            | 件数      |
| -------- | ------ | ------------------------------- | --------- |
| g-actual | 54308  | GenBank 由来（DDBJ 自極分含む） | ~1,369 万 |
| e-actual | 54309  | EMBL 由来                       | ~1.06 億  |
| w-actual | 54310  | WGS                             | ~6,726 万 |

- 接続: `postgresql://guestuser:guestuser@a012:{port}/{dbname}`

##### status

`manager` テーブルの `status` カラム（smallint）で管理。enum 定義は `status` テーブル（`ddbj_table='manager'`, `ddbj_field='status'`）にある。

| st_id | st_name      | g-actual   | e-actual    | w-actual   | 合計        |
| ----- | ------------ | ---------- | ----------- | ---------- | ----------- |
| 1001  | private      | 79,001     | 827,781     | 1,820,562  | 2,727,344   |
| 1002  | public       | 13,431,738 | 104,495,310 | 58,688,598 | 176,615,646 |
| 1004  | suppressed   | 15,429     | 142,970     | 5,302,425  | 5,460,824   |
| 1005  | secondary    | 9,729      | 506         | 16,488     | 26,723      |
| 1006  | killed       | 123,384    | 307,847     | 1,354,087  | 1,785,318   |
| 1007  | unregistered | 31,466     | 225,912     | 80,614     | 337,992     |

`suppresskill_list` テーブルにも suppressed (1004) / secondary (1005) の accession 一覧がある（`manager` テーブルとの件数に微小な差がある）。

##### relation

DAG edges:

- Trad accession -> BioProject
- Trad accession -> BioSample
- Trad accession -> DRR (SRA Run)

`accession` → `link_pr_ac` → `project` テーブルの JOIN で BioProject / BioSample（/ DRR）への紐づけが取れる。`project.project_id` に `PRJDB*` や `SAMD*`, `DRR*` などが入っている。

##### submission_stage

`dataflow` テーブルが accession 単位の作業ログを記録しており、最新レコードの status から `submission_stage` を導出できる。ただし、全 accession に `dataflow` レコードがあるとは限らないため、`submission_stage` は nullable。1 accession あたり最大 276 件のレコードが存在する細粒度なログ。

`dataflow` テーブル:

| カラム   | 型        | 説明                                             |
| -------- | --------- | ------------------------------------------------ |
| `ac_id`  | bigint    | accession ID（`accession` テーブルの FK）        |
| `df_id`  | bigint    | フロー連番                                       |
| `status` | smallint  | ステータスコード（43 種、`status` テーブル参照） |
| `worker` | char      | 作業者                                           |
| `m_date` | timestamp | 操作日時                                         |
| `dt_id`  | bigint    | 詳細 ID（`df_detail` テーブルの FK）             |

主要な status code のグループ分け:

| status code group            | 意味                            | → submission_stage   |
| ---------------------------- | ------------------------------- | -------------------- |
| 1001-1003                    | received / acknowledged         | `submitted`          |
| 1007-1011                    | annotation / review             | `in_curation`        |
| 1012                         | sent to submitter（差し戻し）   | `revision_requested` |
| 1030                         | return from reviewer            | `in_curation`        |
| 1033, 1036, 1004, 1005       | all done / ready for release    | `accepted`           |
| 1050, 1051, 1052, 1053, 1054 | suppress / kill / unregister 等 | （record_status 側） |

`newdata_flow` テーブルで D-way の submission ID（`SUB*`）と g-actual の accession を対応付けできる:

| カラム       | 型      | 説明                                               |
| ------------ | ------- | -------------------------------------------------- |
| `nf_id`      | bigint  | フロー ID                                          |
| `sb_id`      | bigint  | `submission` テーブルの FK                         |
| `sub_name`   | varchar | D-way submission ID（例: `SUB00274318`）           |
| `accessions` | varchar | 割り当て accession 範囲（例: `LC922287-LC922290`） |

##### 主要テーブル

| テーブル            | 用途                                                                      |
| ------------------- | ------------------------------------------------------------------------- |
| `manager`           | accession ごとの status, 各種日付（hold_date, accept_date, open_date 等） |
| `accession`         | ac_id ↔ accession 文字列のマッピング                                      |
| `link_pr_ac`        | accession → project の紐づけ（ac_id, pr_id）                              |
| `project`           | pr_id ↔ project_id（BioProject/BioSample 等）のマッピング                 |
| `suppresskill_list` | suppressed / secondary の accession 一覧（accession, m_date, status）     |
| `status`            | 全テーブル共通の enum 定義マスタ                                          |
| `dataflow`          | accession 単位の作業ログ（submission_stage のソース）                     |
| `newdata_flow`      | D-way submission ID (SUB\*) と accession の紐づけ                         |

#### 他極データ

3ヶ月に一度の定期リリース処理で、GenBank (NCBI) / EMBL (EBI) からデータを取得し、PostgreSQL (g-actual, e-actual) への取り込みと DDBJ 形式 flatfile への変換が行われる。変換前の raw ファイルの所在は不明。

変換後の成果物は `/usr/local/resources/trad` に配置される:

| ディレクトリ | 内容                                                    |
| ------------ | ------------------------------------------------------- |
| `ddbj/`      | リリース版 flatfile（例: release 140）、accession index |
| `ddbjnew/`   | リリース間の新規追加分 flatfile                         |
| `wgs/`       | WGS データ（prefix ごと + `WGS_ORGANISM_LIST.txt`）     |
| `tsa/`       | TSA データ（prefix ごと + `TSA_ORGANISM_LIST.txt`）     |
| `tls/`       | TLS データ（prefix ごと + `TLS_ORGANISM_LIST.txt`）     |
| `tpa/`       | TPA データ                                              |
| `dad/`       | DAD（アミノ酸配列）                                     |

ORGANISM_LIST は TSV で、accession, BioProject, BioSample, SRA 等の紐づけを含む。[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) で relation 構築に使用されている。

### BioProject

accession prefix: `PRJDB*`（自極）, `PRJNA*`（NCBI）, `PRJEB*`（EBI）。正規表現: `^PRJ[DEN][A-Z]\d+$`

#### 自極データ

PostgreSQL `@a011:54306` DB 名 `bioproject`、スキーマ `mass`。

- 接続: `postgresql://guestuser:guestuser@a011:54306/bioproject`

##### status

`mass.project` テーブルの `status_id` カラムで管理。

| status_id | 意味                   | 件数   |
| --------- | ---------------------- | ------ |
| 5100      | submitted              | 23     |
| 5200      | curating               | 91     |
| 5400      | private                | 19,530 |
| 5500      | public                 | 20,228 |
| 5600      | killed                 | 1      |
| 5700      | canceled               | 1,546  |
| 5800      | suppressed             | 58     |
| 5900      | temporarily_suppressed | 97     |

`mass.submission` テーブルにも `status_id` がある。D-way フォーム入力ウィザードの進捗ステップを表す:

| status_id | 件数   | 意味                                                                      |
| --------- | ------ | ------------------------------------------------------------------------- |
| 100       | 3,985  | Created（submission 作成直後、フォーム未入力）                            |
| 200       | 1,517  | Step 1「Submitter 情報」入力完了                                          |
| 300       | 153    | Step 2「General Info」入力完了                                            |
| 400       | 184    | Step 3「Project Type」入力完了                                            |
| 500       | 58     | Step 4-5「Target」「Publication」入力完了                                 |
| 600       | 415    | Step 6「Overview」表示済み。全フォーム完了だが Apply 未完了               |
| 700       | 41,599 | Apply 完了。XML 挿入・accession 発行済み。以降は project.status_id で管理 |
| 750       | 10     | Apply 時の LocusTagPrefix Reserve Error で停止（2014-2017 年のバグ由来）  |

ステータスマスターテーブルは存在せず、意味はアプリケーションコードに埋め込まれている。`form_status_flags` (char(6)) で各ステップの検証状態（0=未入力, 1=OK, 9=エラー）を管理。

##### relation

DAG edges:

- BioProject (umbrella) -> BioProject (child)
- BioProject -> DRP (SRA Study、xref 経由)
- BioProject -> NBDC hum-id
- BioProject -> GEO

PostgreSQL 内:

- `mass.umbrella_info`: BioProject 間の umbrella（親子）関連（1,283 件）
  - `submission_id` → `parent_submission_id`（自極 PSUB 間）
  - `other_parent_project_id`（他極の親プロジェクト ID がある場合）
- `mass.submission_data` の `xref_db` + `xref_id`: 外部 DB との紐づけ（現状は SRA → DRP のみ）

BioProject XML から（[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) `dblink/bioproject.py` で処理）:

- Umbrella 関連（親子）: `<ProjectLinks><Link><Hierarchical type="TopAdmin">` の `<ProjectIDRef accession>` (parent) と `<MemberID accession>` (child) から構築。`umbrella.duckdb` の `umbrella_relation` テーブルに保存
- hum-id 関連: `<ProjectID><LocalID submission_id="humXXXX">` から BioProject → NBDC hum-id。バージョン情報（例: `hum0001.v2`）は `normalize_hum_id()` で除去
- GEO 関連: `<ProjectID><CenterID center="GEO">` から BioProject → GEO

##### 主要テーブル

| テーブル               | 用途                                                             |
| ---------------------- | ---------------------------------------------------------------- |
| `mass.project`         | accession（prefix + counter）, status_id, 各種日付, project_type |
| `mass.submission`      | submission_id ↔ submitter_id, status_id                          |
| `mass.umbrella_info`   | BioProject 間の umbrella（親子）関連                             |
| `mass.submission_data` | EAV 形式の属性データ（xref_db/xref_id 等、92 種の data_name）    |
| `mass.status_history`  | status 変更履歴                                                  |
| `mass.action_history`  | 操作履歴（submission_id, action, action_date）                   |
| `mass.xml`             | submission ごとの XML 本体（バージョン管理付き）                 |

##### XML

| ファイル                                                   | サイズ | 内容                         |
| ---------------------------------------------------------- | ------ | ---------------------------- |
| `/usr/local/resources/bioproject/ddbj_core_bioproject.xml` | 33 MB  | DDBJ 自極分の BioProject XML |
| `/usr/local/resources/bioproject/ddbj_summary.txt`         | 1.9 MB | DDBJ 自極分のサマリ TSV      |

##### livelist

日次で生成。`/lustre9/open/archive/tape/ddbj-dbt/bp-collab/bioproject/` に配置。

| ファイル                                  | 形式                               | 件数（2026/03/11 時点） |
| ----------------------------------------- | ---------------------------------- | ----------------------- |
| `YYYYMMDD.bioproject.ddbj.public.txt`     | TSV (`Accession\tUpdated\tStatus`) | ~20,225                 |
| `YYYYMMDD.bioproject.ddbj.suppressed.txt` | 同上                               | ~155                    |
| `YYYYMMDD.bioproject.ddbj.withdrawn.txt`  | 同上                               | 1                       |
| `YYYYMMDD.bioproject.dump.ddbj.xml`       | XML                                | DDBJ 自極分の dump      |

public + suppressed + withdrawn で ~20,000 件。

#### 他極データ

##### BioProject XML

| ファイル                                         | サイズ | 内容                                   |
| ------------------------------------------------ | ------ | -------------------------------------- |
| `/usr/local/resources/bioproject/bioproject.xml` | 3.6 GB | 全極の BioProject XML（NCBI/EBI/DDBJ） |

`bioproject.xml` は `<PackageSet>` → `<Package>` の繰り返しで、各 Package 内に accession, 日付, umbrella 関連, submission 情報等を含む。

### BioSample

accession prefix: `SAMD*`（自極）。正規表現: TODO（他極含む）。

#### 自極データ

PostgreSQL `@a011:54306` DB 名 `biosample`、スキーマ `mass`。BioProject と同じホスト・ポート。

- 接続: `postgresql://guestuser:guestuser@a011:54306/biosample`
- 自極サンプル数: ~187 万件

##### status

`mass.sample` テーブルの `status_id` カラムで管理。BioProject と同じ enum 体系。

| status_id | 意味                   | 件数    |
| --------- | ---------------------- | ------- |
| 5100      | submitted              | 4,718   |
| 5200      | curating               | 11,239  |
| 5400      | private                | 926,917 |
| 5500      | public                 | 837,644 |
| 5600      | killed                 | 39      |
| 5700      | canceled               | 84,554  |
| 5800      | suppressed             | 2,299   |
| 5900      | temporarily_suppressed | 4,103   |

##### relation

DAG edges:

- BioSample -> BioProject
- BioSample -> DRS (SRA Sample)

PostgreSQL 内:

- `mass.ext_ref`: 外部 DB との紐づけ（smp_id, ext_id, type）。現状は SRA (DRS) のみ 16,237 件

BioSample XML から（[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) `dblink/bp_bs.py` で処理）:

- NCBI XML: `<Link target="bioproject">` または `<Attribute bioproject_accession>` → BioSample -> BioProject
- DDBJ XML: `<Attribute attribute_name="bioproject_id">` または `<Attribute attribute_name="bioproject_accession">` → BioSample -> BioProject

SRA/DRA Accessions.tab から:

- `BioSample` + `BioProject` カラムの DISTINCT ペアから BioSample -> BioProject を抽出（`sra_accessions_tab.py` の `iter_bp_bs_relations()`）
- BioSample/BioProject が数値 ID の場合、XML 由来のマッピング TSV（`bs_id_to_accession.tsv`, `bp_id_to_accession.tsv`）で accession に変換（`dblink/utils.py` の `convert_id_if_needed()`）

##### 主要テーブル

| テーブル          | 用途                                                                  |
| ----------------- | --------------------------------------------------------------------- |
| `mass.sample`     | サンプル本体（status_id, 各種日付, package 情報）                     |
| `mass.accession`  | smp_id ↔ accession_id（`SAMD*`）のマッピング                          |
| `mass.submission` | submission_id ↔ submitter_id                                          |
| `mass.attribute`  | サンプル属性の EAV テーブル（organism, taxonomy_id, geo_loc_name 等） |
| `mass.ext_ref`    | 外部参照（SRA → DRS）                                                 |
| `mass.xml`        | submission ごとの XML 本体                                            |

##### XML

| ファイル                                                   | サイズ     | 内容                        |
| ---------------------------------------------------------- | ---------- | --------------------------- |
| `/usr/local/resources/biosample/ddbj_biosample_set.xml.gz` | 31 MB (gz) | DDBJ 自極分の BioSample XML |
| `/usr/local/resources/biosample/ddbj_summary.txt`          | 29 MB      | DDBJ 自極分のサマリ TSV     |

##### livelist

日次で生成。`/lustre9/open/archive/tape/ddbj-dbt/bs-collab/biosample/` に配置。

| ファイル                                 | 形式                               | 件数（2026/03/11 時点） |
| ---------------------------------------- | ---------------------------------- | ----------------------- |
| `YYYYMMDD.biosample.ddbj.public.txt`     | TSV (`Accession\tUpdated\tStatus`) | ~837,508                |
| `YYYYMMDD.biosample.ddbj.suppressed.txt` | 同上                               | ~6,402                  |
| `YYYYMMDD.biosample.ddbj.withdrawn.txt`  | 同上                               | 39                      |
| `YYYYMMDD.biosample.dump.ddbj.xml`       | XML                                | DDBJ 自極分の dump      |

public + suppressed + withdrawn で ~843,949 件。

#### 他極データ

##### BioSample XML

| ファイル                                              | サイズ      | 内容                 |
| ----------------------------------------------------- | ----------- | -------------------- |
| `/usr/local/resources/biosample/biosample_set.xml.gz` | 4.3 GB (gz) | 全極の BioSample XML |

全極 XML は圧縮状態で 4.3 GB（展開後は 100 GB オーダー）。

### SRA (DRA)

6 種類の accession type を持つ。prefix の 1 文字目が極を表す（D=DDBJ, S=NCBI, E=EBI）。

| type       | prefix パターン | 例                   |
| ---------- | --------------- | -------------------- |
| submission | `[SDE]RA\d+`    | DRA000001, SRA123456 |
| study      | `[SDE]RP\d+`    | DRP000001, SRP123456 |
| experiment | `[SDE]RX\d+`    | DRX000001            |
| run        | `[SDE]RR\d+`    | DRR000001            |
| sample     | `[SDE]RS\d+`    | DRS000001            |
| analysis   | `[SDE]RZ\d+`    | DRZ000001            |

#### 自極データ

##### drmdb（D-way 管理 DB）

PostgreSQL `@a011:54306` DB 名 `drmdb`、スキーマ `mass`。D-way (tracesys) が DRA の submission ワークフローを管理する DB。DRA_Accessions.tab は drmdb から生成される公開済みデータのスナップショット。

- 接続: `postgresql://guestuser:guestuser@a011:54306/drmdb`

###### submission_stage

`mass.submission` テーブルには status カラムがなく、`mass.status_history` テーブルが event sourcing 形式で status 遷移を記録する。最新レコード（`date DESC, status DESC` で rank=1）が現在の status。ステータスマスタテーブルは存在せず、コードはアプリケーション埋め込み。

status code 一覧:

| status | 意味               | 現在件数 | → submission_stage   |
| ------ | ------------------ | -------- | -------------------- |
| 100    | draft              | 26,581   | `draft`              |
| 190    | canceled           | 3        | null                 |
| 300    | submitted          | 710      | `submitted`          |
| 380    | validated          | 4        | `in_curation`        |
| 390    | revision requested | 97       | `revision_requested` |
| 400    | processing         | 53       | `in_curation`        |
| 500    | accessioned        | 3        | `accepted`           |
| 700    | held               | 3,291    | `accepted`           |
| 750    | pre-release        | （※）    | `accepted`           |
| 770    | suppressed         | 117      | `accepted`           |
| 800    | public             | 22,772   | `accepted`           |
| 1000   | withdrawn          | 3,124    | `accepted`           |
| 1100   | intermediate       | 46       | `accepted`           |
| 1200   | intermediate       | 4        | `accepted`           |

（※）750 は 700 → 800 の中間状態であり、最新ステータスとしてはカウントされない。

典型的なワークフロー遷移:

```mermaid
stateDiagram-v2
    100 --> 300: submit
    100 --> 190: cancel
    300 --> 380: validate
    300 --> 390: revision request
    390 --> 300: resubmit
    380 --> 400: process
    400 --> 500: accession
    500 --> 700: hold
    700 --> 750: pre-release
    750 --> 800: publish
    800 --> 770: suppress
    800 --> 1000: withdraw
```

差し戻しループ: 300 ⇄ 390 は複数回発生する場合がある。200 / 600 は初期（2009-2010）のみ使用された旧ステータス。

###### 主要テーブル

| テーブル                  | 用途                                                                |
| ------------------------- | ------------------------------------------------------------------- |
| `mass.submission`         | submission 本体（sub_id, submitter_id, create/submit/hold_date 等） |
| `mass.status_history`     | status 変更履歴（event sourcing、sub_id + status + date）           |
| `mass.submission_group`   | submission のバージョン管理                                         |
| `mass.accession_entity`   | accession 本体（DRA/DRX/DRR/DRS/DRP/DRZ、is_delete/was_public）     |
| `mass.accession_relation` | SRA 内部の accession 間親子関係                                     |
| `mass.ext_entity`         | 外部参照（PSUB/SSUB との紐づけ）                                    |
| `mass.ext_relation`       | 外部エンティティとの関係                                            |
| `mass.meta_entity`        | メタデータ XML                                                      |

###### 主要ビュー

| ビュー                                   | 用途                                                     |
| ---------------------------------------- | -------------------------------------------------------- |
| `mass.dra_summary`                       | submission + 最新 status + accession + BioProject の結合 |
| `mass.current_dra_submission_group_view` | 最新バージョンの submission_group                        |
| `mass.dra_accession_view`                | accession 一覧                                           |
| `mass.dra_accession_meta_view`           | accession + メタデータ XML の結合                        |

##### DRA XML

submission ごとに XML ファイルが配置されている:

```text
/usr/local/resources/dra/fastq/{DRA000}/{DRA000XXX}/
  ├── DRA000XXX.submission.xml
  ├── DRA000XXX.study.xml
  ├── DRA000XXX.experiment.xml
  ├── DRA000XXX.run.xml
  └── DRA000XXX.sample.xml
```

##### DRA_Metadata.tar

[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) が構築する。DRA_Accessions.duckdb から全 submission を列挙 → 各 submission ディレクトリの XML を tar 化。

| tar                | 出力先                  | 入力ソース                                                             |
| ------------------ | ----------------------- | ---------------------------------------------------------------------- |
| `DRA_Metadata.tar` | `{result_dir}/sra_tar/` | DRA XML（`/usr/local/resources/dra/fastq/{DRA000}/{DRA000XXX}/*.xml`） |

tar 内部の構造: `{accession}/{accession}.{type}.xml`。CLI: `sync_dra_tar`

#### status

Accessions.tab の `Status` カラムで管理。SRA（NCBI）と DRA（DDBJ）で status 値が異なる。

**SRA（全極、NCBI 作成）** — ~1.44 億行、30 GB:

| status      | 件数        |
| ----------- | ----------- |
| live        | 127,775,189 |
| unpublished | 11,166,969  |
| suppressed  | 5,147,698   |
| withdrawn   | 2,687       |

ファイル: `/lustre9/open/database/ddbj-dbt/dra-private/mirror/SRA_Accessions/{YYYY}/{MM}/SRA_Accessions.tab.{YYYYMMDD}`

**DRA（自極、DDBJ 作成）** — ~230 万行、422 MB:

| status     | 件数      |
| ---------- | --------- |
| public     | 2,287,939 |
| suppressed | 16,984    |
| withdrawn  | 327       |

ファイル: `/lustre9/open/database/ddbj-dbt/dra-private/tracesys/batch/logs/livelist/ReleaseData/public/{YYYYMMDD}.DRA_Accessions.tab`

注: SRA は `live` / `unpublished`、DRA は `public` と表記が異なる。

#### relation

DAG edges（SRA 内部）:

- Submission -> Study
- Submission -> Analysis（Study が空の場合の fallback）
- Study -> Experiment
- Study -> Analysis
- Experiment -> Run
- Experiment -> Sample
- Run -> Sample

DAG edges（cross-repository）:

- BioProject -> Study
- BioSample -> Sample

Accessions.tab のカラムから構築。各行に `Type`, `Submission`, `Study`, `Experiment`, `Sample`, `BioProject`, `BioSample` が含まれており、これらを組み合わせて relation を抽出する。

SRA 内部 relation:

- Submission -> Study（Type=STUDY）
- Study -> Experiment（Type=EXPERIMENT）
- Study -> Analysis（Type=ANALYSIS）
- Experiment -> Run（Type=RUN）
- Experiment -> Sample（Type=EXPERIMENT）
- Run -> Sample（Type=RUN）

BioProject / BioSample との relation（Accessions.tab に非正規化されたカラムあり）:

- BioProject -> Study / Experiment / Run / Analysis
- BioSample -> Sample / Experiment / Run / Analysis

#### Accessions.tab カラム構造

```text
Accession  Submission  Status  Updated  Published  Received
Type  Center  Visibility  Alias  Experiment  Sample  Study
Loaded  Spots  Bases  Md5sum  BioSample  BioProject  ReplacedBy
```

SRA と DRA で同一スキーマ（20 カラム）。

#### 他極データ

- SRA_Accessions.tab: 日次で NCBI FTP から SCP（全極の status / relation ソース）
- SRA_Run_Members.tab: 同上
- NCBI SRA Metadata XML: `/lustre9/open/database/ddbj-dbt/dra-private/mirror/Metadata/Metadata/`（2014 年で更新停止）
- suppressed / kill list: NCBI から非統一形式（ファイル、ticket、mail）で不定期に送付

##### NCBI_SRA_Metadata.tar

[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) が構築する。JSONL 生成時に tar 内の XML を読み込む。

| tar                     | 出力先                  | 入力ソース                                                                            |
| ----------------------- | ----------------------- | ------------------------------------------------------------------------------------- |
| `NCBI_SRA_Metadata.tar` | `{result_dir}/sra_tar/` | NCBI FTP の Full/Daily tar.gz（`https://ftp.ncbi.nlm.nih.gov/sra/reports/Metadata/`） |

tar 内部の構造: `{accession}/{accession}.{type}.xml`。Full tar.gz をダウンロード・展開し、以降は Daily tar.gz を append（`tar -Af`）で増分追記。CLI: `sync_ncbi_tar`

## DDBJ 独自リポジトリ

### GEA

accession パターン: `^E-GEAD-\d+$`。他極データなし。~682 件。

#### データソース

##### dordb（D-way 管理 DB）

PostgreSQL `@a011:54306` DB 名 `dordb`、スキーマ `mass`。D-way (tracesys) が GEA の submission ワークフローを管理する DB。livelist は dordb から生成される公開済みデータのスナップショット。

- 接続: `postgresql://guestuser:guestuser@a011:54306/dordb`

2 レベルの status 構造を持つ:

- `submission_status_history`: submission 単位のワークフロー状態（record-idm の `submission_stage` ソース）
- `object_status_history`: accession 単位の公開状態（record-idm の `record_status` の補助ソース）

###### submission_status

`mass.submission_status_history` テーブルで submission 単位のワークフロー状態を記録。`mass.current_submission_status` ビューで最新状態を取得できる。ステータスマスタテーブルは存在しない。

submission_status_type 一覧:

| status | 意味               | → submission_stage   | 備考               |
| ------ | ------------------ | -------------------- | ------------------ |
| 0      | created            | `draft`              |                    |
| 10     | data submitted     | `submitted`          |                    |
| 20     | preparing          | `submitted`          | 自動処理           |
| 30     | transferring       | `submitted`          | 自動処理           |
| 40     | scanning           | `submitted`          | 自動処理           |
| 50     | validating         | `submitted`          | 自動処理           |
| 60     | loading            | `submitted`          | 自動処理           |
| 100    | preparation error  | `revision_requested` |                    |
| 110    | scanning error     | `revision_requested` |                    |
| 120    | validation error   | `revision_requested` |                    |
| 130    | transfer error     | `revision_requested` |                    |
| 200    | data validated     | `in_curation`        |                    |
| 210    | curating           | `in_curation`        |                    |
| 220    | accession assigned | `accepted`           |                    |
| 230    | metadata loaded    | `accepted`           |                    |
| 240    | data loading       | `accepted`           |                    |
| 250    | data error         | `revision_requested` | データロードエラー |
| 260    | private            | `accepted`           | hold 期間中        |
| 270    | wait for release   | `accepted`           | 公開待ち           |
| 300    | public             | `accepted`           |                    |
| 400    | canceled           | null                 |                    |
| 410    | suppressed         | `accepted`           |                    |
| 420    | killed             | `accepted`           |                    |

典型的なワークフロー遷移:

```mermaid
stateDiagram-v2
    0 --> 10: submit
    10 --> 20: prepare
    20 --> 30: transfer
    30 --> 40: scan
    40 --> 50: validate
    50 --> 60: load
    60 --> 200: validated
    200 --> 210: curate
    210 --> 220: accession
    220 --> 230: metadata
    230 --> 240: data load
    240 --> 260: private
    260 --> 270: wait release
    270 --> 300: publish
    300 --> 410: suppress
    300 --> 420: kill
    20 --> 100: prep error
    40 --> 110: scan error
    50 --> 120: validation error
    60 --> 130: transfer error
    230 --> 250: data error
```

エラー系（100-130, 250）は差し戻し後に再投稿可能。400 (canceled) は多くの状態から遷移可能。

###### object_status

`mass.object_status_history` テーブルで accession 単位の公開状態を記録。後半のステータス（260-420）のみ。

| status | 意味             | 件数   |
| ------ | ---------------- | ------ |
| 260    | private          | 38,929 |
| 270    | wait for release | 24,678 |
| 300    | public           | 5,605  |
| 400    | canceled         | 923    |
| 410    | suppressed       | 81     |
| 420    | killed           | 17     |

###### submission_type / accession_type

submission_type:

| type | 件数  | 意味         |
| ---- | ----- | ------------ |
| 1    | 635   | Microarray   |
| 2    | 1,532 | Sequencing   |
| 3    | 9     | Array Design |
| 4    | 9     | Update       |

accession_type:

| type | prefix    | 件数   | 意味                       |
| ---- | --------- | ------ | -------------------------- |
| 1    | E-GEAD-   | 1,063  | GEA Experiment（本体）     |
| 2    | P-GEAD-   | 6,266  | Protocol                   |
| 3    | A-GEA- 等 | 316    | Array Design               |
| 7    | DRX       | 7,196  | DRA Experiment（外部参照） |
| 9    | DRR       | 7,403  | DRA Run（外部参照）        |
| 11   | PRJDB     | 840    | BioProject（外部参照）     |
| 12   | SAMD      | 15,883 | BioSample（外部参照）      |
| 18   | CBX       | 254    | CIBEX（レガシー）          |

###### 主要テーブル

| テーブル                         | 用途                              |
| -------------------------------- | --------------------------------- |
| `mass.submission`                | submission 本体                   |
| `mass.submission_status_history` | submission 単位の status 変更履歴 |
| `mass.accession`                 | accession 本体                    |
| `mass.object_status_history`     | accession 単位の status 変更履歴  |
| `mass.object_group`              | オブジェクトグループ              |
| `mass.relation`                  | GEA 内部の relation               |
| `mass.secondary_relation`        | 二次的な relation                 |
| `mass.metadata`                  | メタデータ                        |
| `mass.file`                      | ファイル管理                      |

###### 主要ビュー

| ビュー                           | 用途                     |
| -------------------------------- | ------------------------ |
| `mass.current_submission_status` | 最新の submission status |
| `mass.current_object_status`     | 最新の object status     |
| `mass.current_metadata`          | 最新のメタデータ         |

##### IDF/SDRF ファイル

ベースパス: `/usr/local/resources/gea/experiment/`

```text
E-GEAD-{N000}/
  E-GEAD-{NNNN}/
    E-GEAD-{NNNN}.idf.txt
    E-GEAD-{NNNN}.sdrf.txt
    E-GEAD-{NNNN}.filelist.txt
    E-GEAD-{NNNN}.processed.zip
```

##### livelist

livelist ファイル（`/usr/local/resources/gea/experiment/livelist.txt`）で管理。公開済みデータのみ。TSV 形式。

| カラム    | 内容                                        |
| --------- | ------------------------------------------- |
| Accession | E-GEAD-{N}                                  |
| Status    | Public / Permanently Suppressed / Withdrawn |
| Updated   | ISO 8601                                    |
| Published | ISO 8601                                    |
| Submitted | ISO 8601                                    |
| Type      | Experiment                                  |

| status                 | 件数 |
| ---------------------- | ---- |
| Public                 | 676  |
| Permanently Suppressed | 4    |
| Withdrawn              | 2    |

##### relation

DAG edges:

- GEA -> BioProject
- GEA -> BioSample
- GEA -> SRA Experiment
- GEA -> SRA Run

IDF / SDRF ファイルから構築:

- IDF の `Comment[BioProject]` → GEA -> BioProject
- SDRF の `Comment[BioSample]` カラム → GEA -> BioSample
- SDRF の `Comment[SRA_EXPERIMENT]`, `Comment[SRA_RUN]` → GEA -> SRA Experiment / Run

### JGA

制限公開（controlled-access）のヒトデータ。他極データなし。organism は常に Homo sapiens。

accession パターン:

| type       | prefix    | 例         | 備考                      |
| ---------- | --------- | ---------- | ------------------------- |
| submission | `JGA\d+`  | JGA000123  | 審査後に発行される公開 ID |
| study      | `JGAS\d+` | JGAS000001 |                           |
| dataset    | `JGAD\d+` | JGAD000053 |                           |
| sample     | `JGAN\d+` | JGAN000001 |                           |
| experiment | `JGAX\d+` | JGAX000001 |                           |
| data (run) | `JGAR\d+` | JGAR000001 |                           |
| analysis   | `JGAZ\d+` | JGAZ000001 |                           |
| dac        | `JGAC\d+` | JGAC000001 | Data Access Committee     |
| policy     | `JGAP\d+` | JGAP000002 |                           |

申請管理上の内部 ID:

| type            | prefix    | 例         | 備考                 |
| --------------- | --------- | ---------- | -------------------- |
| 内部 submission | `JSUB\d+` | JSUB000481 | 登録時に発行。非公開 |
| データ提供申請  | `J-DS\d+` | J-DS002504 |                      |
| データ利用申請  | `J-DU\d+` | J-DU006529 |                      |

[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) で accession type として定義されているのは study, dataset, dac, policy の 4 種。

#### 申請管理システム

個人ゲノム解析区画の VM 上に PostgreSQL ベースの「申請管理システム」が稼働している。XML / CSV は、このシステムから日次バッチで出力される。

- 詳細: `~/git/github.com/dbcls/humandbs/apps/backend/jga-shinsei`
- DB: PostgreSQL（スキーマ `ts_jgasys`、DB 名 `jgadb`）
- 認証: DDBJ Account（Keycloak OIDC、`idp-staging.ddbj.nig.ac.jp`）

申請ステータス（`appl_status_type`）:

| コード | 意味                                    |
| ------ | --------------------------------------- |
| 10     | 申請書類作成中 (Preparing)              |
| 20     | 申請完了 (Submitted)                    |
| 30     | 申請者へ差し戻し中 (Revision requested) |
| 40     | 審査中 (DAC reviewing)                  |
| 50     | 申請却下 (Rejected)                     |
| 60     | 申請承認 (Approved)                     |
| 70     | 申請取り下げ (Canceled)                 |
| 80     | 利用期間終了 (Closed)                   |

#### データソース

XML + CSV ファイルがソース（申請管理システムの PostgreSQL から日次バッチで出力）。

ベースパス: `/usr/local/shared_data/jga/metadata-history/metadata/`

##### XML ファイル

| ファイル           | 行数       |
| ------------------ | ---------- |
| jga-study.xml      | 37,876     |
| jga-dataset.xml    | 442,194    |
| jga-experiment.xml | 16,194,502 |
| jga-analysis.xml   | 5,824,058  |
| jga-data.xml       | 5,343,707  |
| jga-dac.xml        | 19         |
| jga-policy.xml     | 199        |

##### status

[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) では常に `"live"` 固定。動的な status 管理の仕組みがない。

##### relation

DAG edges（JGA 内部）:

- study -> experiment -> data (run)
- study -> analysis
- dataset -> data
- dataset -> analysis
- dataset -> policy -> dac
- experiment -> sample
- analysis -> sample
- analysis -> data

DAG edges（cross-repository / 外部参照）:

- study -> NBDC hum-id
- study -> PubMed

CSV ファイルで管理。形式: `entry_id,self_accession,parent_accession`

| CSV ファイル                   | 件数       | relation             |
| ------------------------------ | ---------- | -------------------- |
| dataset-analysis-relation.csv  | ~82,709    | dataset -> analysis  |
| dataset-data-relation.csv      | ~352,308   | dataset -> data      |
| dataset-policy-relation.csv    | ~665       | dataset -> policy    |
| policy-dac-relation.csv        | ~542       | policy -> dac        |
| analysis-study-relation.csv    | ~82,466    | study -> analysis    |
| data-experiment-relation.csv   | ~352,307   | experiment -> data   |
| experiment-study-relation.csv  | ~350,365   | study -> experiment  |
| experiment-sample-relation.csv | ~350,365   | experiment -> sample |
| analysis-sample-relation.csv   | ~3,625,181 | analysis -> sample   |
| analysis-data-relation.csv     | ~3,692     | analysis -> data     |

[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) では CSV の JOIN で推移的な relation を構築:

- dataset -> study（dataset-analysis + analysis-study、または dataset-data + data-experiment + experiment-study）
- dataset -> dac（dataset-policy + policy-dac）
- study -> dac, study -> policy（上記の逆引き + JOIN）

XML から:

- study -> hum-id（NBDC Number）: `STUDY_ATTRIBUTES/STUDY_ATTRIBUTE[@TAG="NBDC Number"]`
- study -> pubmed-id: `PUBLICATIONS/PUBLICATION[@DB_TYPE="PUBMED"]`

##### 日付データ

CSV ファイルで管理。形式: `accession,datecreated,datepublished,datemodified`

| ファイル            | 件数     |
| ------------------- | -------- |
| study.date.csv      | 531      |
| dataset.date.csv    | 664      |
| policy.date.csv     | 14       |
| dac.date.csv        | 1        |
| experiment.date.csv | ~350,309 |
| sample.date.csv     | ~887,561 |
| analysis.date.csv   | ~82,713  |
| data.date.csv       | ~352,251 |

### AGD

JGA とほぼ同じ構成。申請管理システムは JGA のクローン。

- DB: PostgreSQL（申請管理システム、JGA と同じ構成）
- XML / TSV: 出力されていない

### MetaboBank

メタボロミクスデータリポジトリ。質量分析 (MS)、核磁気共鳴 (NMR)、イメージング MS のデータを扱う。MetaboLights 等とのデータ交換なし。

accession パターン: `^MTBKS\d+$`（study レベルのみ、ゼロ埋めなし）。例: MTBKS1, MTBKS232

#### データソース

PostgreSQL なし。IDF/SDRF ファイル（MAGE-TAB 形式）で管理。GEA と同じメタデータ形式。

ベースパス: `/lustre9/open/shared_data/metabobank/`

```text
study/          IDF/SDRF + raw/processed データ（公開済み）
dataset/        study ごとの zip（156 件）
project/        study ごとのディレクトリ
rdf/            KNApSAcK
```

study ディレクトリは 1 階層のフラット構造（GEA の 2 階層と異なる）:

```text
study/MTBKS232/
  MTBKS232.idf.txt          IDF（BioProject ID、プロトコル、登録者情報）
  MTBKS232.sdrf.txt         SDRF（サンプル・測定・データファイルの関係）
  MTBKS232.positive.maf.txt MAF（代謝物同定結果）
  MTBKS232.negative.maf.txt MAF
  MTBKS232.filelist.txt     ファイル一覧
  raw/                      生データ
  processed/                解析データ
  mzB/                      Reifycs mzB 形式
```

初期の study（MTBKS1〜59 付近）は IDF/SDRF がなく、raw データのみのディレクトリ構成。study 総数は 156 ディレクトリ（うち IDF あり ~109）。

##### status

status file 方式。study ディレクトリ内に配置するファイルの有無で判定する（metabobank_tools による）:

| 判定条件                                    | status                              |
| ------------------------------------------- | ----------------------------------- |
| status file なし + Public Release Date なし | Private                             |
| status file なし + Public Release Date あり | Public                              |
| `status-temporarily-suppressed.txt`         | Temporarily suppressed              |
| `status-permanently-suppressed.txt`         | Permanently suppressed              |
| `status-cancelled.txt`                      | Cancelled                           |
| `status-killed.txt`                         | Killed                              |
| `status-reviewer-access.txt`                | In review（レビュアー向けアクセス） |

livelist は日次 cron で生成。公開用 livelist は TSV 形式（`Accession, Status, Updated, Published, Submitted`）。現在 138 件、全件 Public。

##### relation

DAG edges:

- MetaboBank study -> BioProject
- MetaboBank study -> BioSample

IDF/SDRF から構築（GEA と共通ロジック、[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) の `idf_sdrf.py`）:

- IDF `Comment[BioProject]` → MetaboBank -> BioProject
- SDRF `Comment[BioSample]`（= `Characteristics[biosample_accession]`）→ MetaboBank -> BioSample

#### 管理ツール

`metabobank_tools`（Ruby gem、[GitHub](https://github.com/ddbj/metabobank_tools)）:

| コマンド      | 用途                                    |
| ------------- | --------------------------------------- |
| `mb-validate` | IDF/SDRF/データファイルのバリデーション |
| `mb-public`   | 初回公開・再配布時のファイル処理        |
| `mb-livelist` | livelist 生成（日次 cron）              |
| `mb-filelist` | filelist 生成                           |
| `mb-update`   | IDF の status 更新                      |
| `mb-acc-mail` | accession 通知メール生成                |

登録口: JIRA チケット（`MBS-*`）で管理。Excel テンプレート → IDF/SDRF 変換（`mbexcel2idfsdrf.rb`）。

### JVar

TogoVar-repository（JVar, Japan Variation Database）。ヒトのバリアント、アリル頻度、遺伝子型のための公的データベース。2024 年 9 月ローンチ。ヒトのみが対象。将来的に NCBI dbSNP / dbVar とのデータ交換が予定されている。

2 つのサブカテゴリから構成される:

- TogoVar-repository-SNP: 50bp 以下の SNV/insertion/deletion（dbSNP 相当）
- TogoVar-repository-SV: 50bp 超の構造変異（dbVar 相当）

なお、TogoVar-repository (DDBJ) とは別に、DBCLS が運営する TogoVar（バリアントカタログ）がある。TogoVar は `tgv` ID（例: tgv122011872）を発行し、日本人集団で観測されたバリアントに alternative allele ごとの永続識別子を付与する。dbSNP 未登録の約 5,050 万バリアントにも付与されている。

accession パターン:

| type              | prefix | 正規表現    | 例    | 備考                            |
| ----------------- | ------ | ----------- | ----- | ------------------------------- |
| study             | `dstd` | `^dstd\d+$` | dstd1 | SNP/SV 共通                     |
| SNP variant       | `dss`  | `^dss\d+$`  | dss1  | short variant                   |
| SV variant call   | `dssv` | `^dssv\d+$` | dssv1 | structural variant の個別コール |
| SV variant region | `dsv`  | `^dsv\d+$`  | dsv1  | structural variant のリージョン |

SampleSet, Experiment, Dataset にはアクセッション番号なし（内部連番 ID のみ）。

申請管理上の内部 ID:

| type       | prefix | 正規表現      | 例         | 備考            |
| ---------- | ------ | ------------- | ---------- | --------------- |
| submission | `VSUB` | `^VSUB\d{6}$` | VSUB000001 | 非公開の内部 ID |

関連する外部 ID:

| type | 管理者          | 正規表現   | 例           | 備考                                     |
| ---- | --------------- | ---------- | ------------ | ---------------------------------------- |
| tgv  | DBCLS (TogoVar) | `^tgv\d+$` | tgv122011872 | allele ごとの永続識別子                  |
| rs   | NCBI (dbSNP)    | `^rs\d+$`  | rs1377973775 | refSNP ID                                |
| ss   | NCBI (dbSNP)    | `^ss\d+$`  | —            | submitted SNP ID（dss 取り込み後に発行） |

#### データソース

PostgreSQL なし。完全にファイルベースの手作業運用。

処理ツール（Ruby, Singularity コンテナで運用）:

| ツール                 | 用途                                 |
| ---------------------- | ------------------------------------ |
| `togovar-convert.rb`   | validation + dbSNP/dbVar 形式変換    |
| `togovar-accession.rb` | accession 発行 + 公開用データ作成    |
| `ref-validator.rb`     | リファレンス配列との塩基一致チェック |
| `batch-validation.sh`  | 大サイズ VCF 処理用バッチ            |

ソースコード: [ddbj/togovar-repository](https://github.com/ddbj/togovar-repository)

スパコン上の作業ディレクトリ: `/lustre9/open/home/tf/repos/jvar/`

accession 管理: `study/last.txt`（TSV 形式、`dstd, submission_id, dss, dssv, dsv` の 5 カラム）。accession のラストナンバーを管理し、次の発行番号を決定する。

登録フロー:

1. D-way アカウント作成 + 公開鍵登録
2. TogoVar-repository submission form から申請
3. BioProject / BioSample を事前登録（必須）
4. Excel メタデータ（Study, SampleSet, Sample, Experiment, Dataset, Variant Call/Region）を準備
5. VCF ファイルを scp/sftp でアップロード
6. `togovar-convert.rb` で validation + dbSNP/dbVar 形式変換
7. `togovar-accession.rb` で accession 発行

submission ファイル配置:

```text
submission/VSUB000001/
  VSUB000001_SNP.xlsx          メタデータ Excel（SNP の場合）
  VSUB000001_SV.xlsx           メタデータ Excel（SV の場合）
  submitted/                   登録された VCF ファイル
  accessioned/                 accession 付与後の成果物
    dstd{N}.meta.tsv           公開用メタデータ
    dstd{N}_a{M}.vcf           dbSNP 用 VCF（SNP, assay ごと）
    dstd{N}.dbvar.xml          dbVar 用 XML（SV）
    dstd{N}.variant_call.tsv   SV variant call TSV
    dstd{N}.variant_region.tsv SV variant region TSV
```

##### status

明示的な status enum は定義されていない。Excel メタデータの `Hold/Release` フィールドで制御:

| Hold/Release | 動作                                               |
| ------------ | -------------------------------------------------- |
| Hold         | 非公開（dbVar XML に `hold_date=2040-01-01` 設定） |
| Release      | 即時公開                                           |

Hold の場合、最大 3 年間の保持が可能で延長も可。公開後の suppressed / withdrawn 等の状態遷移は未実装（コード上に存在しない）。

##### relation

DAG edges:

- Study -> BioProject
- Sample -> BioSample
- Study -> Dataset -> Experiment -> SampleSet -> Sample（内部階層）

メタデータ Excel / TSV から構築:

- Study -> BioProject: Excel の `BioProject Accession` フィールド（必須）
- Sample -> BioSample: Excel の `BioSample Accession` フィールド（必須）

SNP の場合、公開後に dbSNP に取り込まれ ss/rs が発行される（次の build で rs にマージ）。SV は dbVar とのデータ交換が予定されている。

#### 公開データ

`/usr/local/shared_data/togovar/` に配置（`/lustre9/open/shared_data/togovar/` にも同一内容）。

現在公開されているのは 2 study のみ:

| study | type | BioProject | 内容                                              |
| ----- | ---- | ---------- | ------------------------------------------------- |
| dstd1 | SV   | PRJDB16199 | 1 sample (NA19079), manta SV call                 |
| dstd2 | SNP  | PRJDB18486 | ~21,000 sample, WGS variant, 23 VCF（染色体ごと） |

```text
/usr/local/shared_data/togovar/
  dstd1/
    dstd1.meta.tsv            メタデータ
    dbvar/                    dbVar XML + genotype
    submitted/                登録 VCF
  dstd2/
    dstd2.meta.tsv            メタデータ
    dbsnp/                    dbSNP 用 VCF（assay ごと, gz 圧縮）
    submitted/                登録 VCF
```

## 参照データ

### NCBI GenBank Assembly

NCBI が管理する genome assembly の情報。assembly_summary_genbank.txt から insdc-assembly / insdc-master と BioProject / BioSample の relation を構築する。DDBJ 独自のデータはない。

#### ID 体系

| type           | 正規表現                                                            | 例                     | 備考                                                       |
| -------------- | ------------------------------------------------------------------- | ---------------------- | ---------------------------------------------------------- |
| insdc-assembly | `^GCA_[0-9]{9}(\.[0-9]+)?$`                                         | GCA_000001215.4        | GenBank Assembly accession。バージョン接尾辞あり           |
| insdc-master   | `^([A-Z]0{5}\|[A-Z]{2}0{6}\|[A-Z]{4,6}0{8,10}\|[A-J][A-Z]{2}0{5})$` | AABU00000000, CP000000 | WGS/TSA/TLS 等の master record。数字部分はゼロ化して正規化 |

insdc-master の正規化例: `CP035466.1` → `CP000000`、`BAAA01000001-1` → `BAAA00000000`（`normalize_master_id()` でバージョン・ハイフン除去後、全数字をゼロ置換）。

#### データソース

**assembly_summary_genbank.txt**（NCBI FTP からストリーミング取得）:

URL: `https://ftp.ncbi.nlm.nih.gov/genomes/ASSEMBLY_REPORTS/assembly_summary_genbank.txt`

TSV 形式。relation 構築に使うカラム:

| カラム | 内容                          |
| ------ | ----------------------------- |
| col[0] | assembly accession（`GCA_*`） |
| col[1] | bioproject accession          |
| col[2] | biosample accession           |
| col[3] | wgs_master accession          |

**TRAD ORGANISM_LIST**（insdc-master の補完ソース）:

| ファイル                                                      | 内容    |
| ------------------------------------------------------------- | ------- |
| `/usr/local/resources/trad/wgs/WGS_ORGANISM_LIST.txt`         | WGS     |
| `/usr/local/resources/trad/tsa/TSA_ORGANISM_LIST.txt`         | TSA     |
| `/usr/local/resources/trad/tls/TLS_ORGANISM_LIST.txt`         | TLS     |
| `/usr/local/resources/trad/tpa/wgs/TPA_WGS_ORGANISM_LIST.txt` | TPA-WGS |
| `/usr/local/resources/trad/tpa/tsa/TPA_TSA_ORGANISM_LIST.txt` | TPA-TSA |
| `/usr/local/resources/trad/tpa/tls/TPA_TLS_ORGANISM_LIST.txt` | TPA-TLS |

col[3]: master accession、col[9]: bioproject、col[10]: biosample。

#### relation

DAG edges:

- insdc-assembly -> bioproject
- insdc-assembly -> biosample
- insdc-assembly -> insdc-master
- insdc-master -> bioproject
- insdc-master -> biosample
- biosample -> bioproject（assembly_summary 由来の補完）

[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) `dblink/assembly_and_master.py` で処理。

### NCBI taxonomy

生物種の分類情報。NCBI が管理する taxonomy データベースから tax dump を日次で取得している。DDBJ 独自のデータはなく、各リポジトリが個別に参照している。

ID: taxonomy ID（数値のみ、例: 9606 = Homo sapiens）。正規表現: `^\d+$`

#### tax dump

NCBI FTP から日次取得。パス: `/lustre9/open/database/ddbj-dbt/dra-private/mirror/taxonomy/dmp/`

| ファイル         | サイズ | 行数       | 内容                                                                       |
| ---------------- | ------ | ---------- | -------------------------------------------------------------------------- |
| `names.dmp`      | 266 MB | ~4,573,106 | taxonomy ID ↔ 名前のマッピング（scientific name, synonym, common name 等） |
| `nodes.dmp`      | 198 MB | ~2,730,579 | taxonomy ノード（parent, rank, division 等の階層構造）                     |
| `delnodes.dmp`   | 8.1 MB | ~863,020   | 削除された taxonomy ID                                                     |
| `merged.dmp`     | 1.8 MB | ~96,801    | 統合された taxonomy ID（old → new のマッピング）                           |
| `citations.dmp`  | 19 MB  | -          | 文献情報                                                                   |
| `division.dmp`   | 452 B  | -          | division マスタ（BCT, PLN, VRT, MAM 等）                                   |
| `gencode.dmp`    | 4.9 KB | -          | 遺伝コード定義                                                             |
| `images.dmp`     | 4.9 MB | -          | 生物種画像情報                                                             |
| `taxdump.tar.gz` | 70 MB  | -          | 上記一式のアーカイブ                                                       |

ファイル形式: BCP-like dump。フィールド区切り `\t|\t`、行区切り `\t|\n`。

`nodes.dmp` の主なカラム:

```text
tax_id | parent_tax_id | rank | embl_code | division_id | ... | GenBank_hidden_flag | hidden_subtree_root_flag | comments
```

`names.dmp` の主なカラム:

```text
tax_id | name_txt | unique_name | name_class
```

name_class: scientific name, synonym, common name, genbank common name, blast name 等。

#### 各リポジトリでの参照方法

| リポジトリ | 参照元             | フィールド                             |
| ---------- | ------------------ | -------------------------------------- |
| BioProject | BioProject XML     | `Organism/taxID`                       |
| BioSample  | BioSample XML / DB | `taxonomy_id` 属性                     |
| SRA (DRA)  | SRA XML            | `SAMPLE/SAMPLE_NAME/TAXON_ID`          |
| Trad       | PostgreSQL         | ORGANISM_LIST.txt に taxonomy 情報含む |
| JGA        | なし               | 固定値（9606 = Homo sapiens）          |
| MetaboBank | SDRF               | `Characteristics[taxonomy_id]` カラム  |

[ddbj-search-converter](https://github.com/ddbj/ddbj-search-converter) では taxonomy ID を `Organism` オブジェクト（identifier + name）として統一的に扱い、JSONL の `organism` フィールドに含めている。DBLink の `AccessionType` にも `"taxonomy"` が定義されているが、dblink で taxonomy との relation を構築する処理は現状未実装（JSONL 出力時に xref URL として使われるのみ）。

#### 備考

- public/private の区分あり（private: 未公開だが生物名は assign 済み）
- 現状は各リポジトリが個別に取り込んでおり、外部参照的に一元管理したい
- taxonomy.ttl: `/lustre9/open/shared_data/rdf/ontologies/taxonomy.ttl`（1.7 GB, ~3,964 万行、日次生成）
- JSON-LD 版のニーズあり
