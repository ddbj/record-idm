# Accession 間 Relation

DDBJ の各リポジトリが管理する accession 間の relation を定義する。
データソースの詳細は [repositories.md](./repositories.md) を参照。

## 凡例

- 図の矢印方向: 概念的な親 → 子（BioProject を頂点とする DAG の上から下）
- 実線（`-->`）: 直接的な参照関係
- 破線（`-.->`）: 間接的・オプショナルな関係
- Edge 一覧テーブルはデータソース上の参照方向で記載（`A -> B` = 「A のデータに B への参照がある」）

## リポジトリ間 Relation

BioProject を頂点として、各リポジトリの accession がどのように紐づくかの概要。

```mermaid
graph TD
    BP_U[umbrella-bioproject] -->|child| BP[bioproject]
    BP --> BS[biosample]

    subgraph SRA
        DRA[sra-submission] --> DRP[sra-study]
        DRA -.-> DRZ[sra-analysis]
        DRP --> DRX[sra-experiment]
        DRP --> DRZ
        DRX --> DRR[sra-run]
        DRX --> DRS[sra-sample]
        DRR --> DRS
    end

    BP --> DRP
    BS --> DRS

    BP -.-> Trad[trad]
    BS -.-> Trad
    Trad -.-> DRR

    BP --> GEA[gea]
    BS --> GEA
    GEA -.-> DRX
    GEA -.-> DRR

    BP --> MB[metabobank]
    BS --> MB

    subgraph JGA
        JGAS[jga-study] --> JGAD[jga-dataset]
        JGAD --> JGAP[jga-policy]
        JGAP --> JGAC[jga-dac]
    end

    JGAS --> hum[hum-id]
    JGAS --> pubmed[pubmed-id]
    BP --> hum
    BP --> geo_id[geo]

    BP -.-> dstd[jvar-study]
    BS -.-> JVSmp[jvar-sample]
    dstd --> JVSmp
```

### 補足

- **JGA/AGD と BioProject**: 直接の edge はない。BioProject と JGA Study がそれぞれ hum-id を参照しており、hum-id を介して間接的に対応する。直接 edge の追加は TODO（[submission.md](./submission.md) 参照）
- **Trad / JVar**: 間接的な参照（破線）。データソースの詳細は Edge 一覧を参照
- **SRA の denormalized edge**: `bioproject -> sra-experiment / sra-run / sra-analysis`、`biosample -> sra-experiment / sra-run / sra-analysis` も存在する（Accessions.tab の非正規化カラム由来）。全体図では省略、SRA 内部図に記載
- **Assembly**: 下記の別図を参照

### Assembly

insdc-assembly / insdc-master は BioProject / BioSample を参照する（全体図の DAG 方向と逆になるため別図）。

```mermaid
graph LR
    GCA[insdc-assembly] --> BP[bioproject]
    GCA --> BS[biosample]
    GCA --> Master[insdc-master]
    Master --> BP
    Master --> BS
```

データソース: `assembly_summary_genbank.txt`（NCBI FTP）+ Trad organism list。

## SRA 内部 Relation

SRA の 6 accession type 間の階層と、BioProject / BioSample との接続。
実線は正規化された親子 edge、破線は Accessions.tab の非正規化カラム由来（denormalized）。

```mermaid
graph TD
    BP[bioproject]
    BS[biosample]

    subgraph SRA
        DRA[sra-submission] --> DRP[sra-study]
        DRA -.->|Study 空時| DRZ[sra-analysis]
        DRP --> DRX[sra-experiment]
        DRP --> DRZ
        DRX --> DRR[sra-run]
        DRX --> DRS[sra-sample]
        DRR --> DRS
    end

    BP --> DRP
    BS --> DRS

    BP -.-> DRX
    BP -.-> DRR
    BP -.-> DRZ
    BS -.-> DRX
    BS -.-> DRR
    BS -.-> DRZ
```

prefix の 1 文字目が極を表す: D=DDBJ, S=NCBI, E=EBI（例: DRR, SRR, ERR）。

## JGA/AGD 内部 Relation

JGA accession type 間の階層と、申請管理システムの ID との接続。AGD も同一構成。

Study（研究単位）と Dataset（データアクセス単位）の 2 つの頂点を持つ DAG。

```mermaid
graph TD
    subgraph 申請管理
        JDS["J-DS<br>(データ提供申請)"]
        JDU["J-DU<br>(データ利用申請)"]
        JSUB["JSUB<br>(内部 submission ID)"]
    end

    JDS -->|submission_permission| JGA_sub["JGA submission<br>(JGA000NNN)"]
    JSUB -.->|alias| JGA_sub
    JDU -->|use_permission| JGAD

    subgraph "研究 (Study)"
        JGAS["jga-study<br>(JGAS)"]
        JGAX["jga-experiment<br>(JGAX)"]
        JGAZ["jga-analysis<br>(JGAZ)"]
    end

    subgraph "公開制御 (Dataset)"
        JGAD["jga-dataset<br>(JGAD)"]
        JGAP["jga-policy<br>(JGAP)"]
        JGAC["jga-dac<br>(JGAC)"]
    end

    JGAS --> JGAX
    JGAS --> JGAZ
    JGAS --> JGAD

    JGAX --> JGAR["jga-data<br>(JGAR)"]
    JGAX --> JGAN["jga-sample<br>(JGAN)"]
    JGAZ --> JGAR
    JGAZ --> JGAN

    JGAD --> JGAR
    JGAD --> JGAZ
    JGAD --> JGAP
    JGAP --> JGAC

    JGAS --> hum["hum-id"]
    JGAS --> pubmed["pubmed-id"]
```

## JVar 内部 Relation

内部階層は accession なし（内部 ID のみ）。accession を持つのは study と variant のみ。

```mermaid
graph TD
    dstd["jvar-study<br>(dstd)"] --> Dataset["Dataset<br>(内部 ID)"]
    Dataset --> Experiment["Experiment<br>(内部 ID)"]
    Experiment --> SampleSet["SampleSet<br>(内部 ID)"]
    SampleSet --> Sample["Sample<br>(内部 ID)"]

    dstd -.-> BP[bioproject]
    Sample -.-> BS[biosample]

    dstd --- dss["SNP Variant (dss)"]
    dstd --- dssv["SV Variant Call (dssv)"]
    dstd --- dsv["SV Variant Region (dsv)"]

    style Dataset fill:#f0f0f0,stroke:#999
    style Experiment fill:#f0f0f0,stroke:#999
    style SampleSet fill:#f0f0f0,stroke:#999
    style Sample fill:#f0f0f0,stroke:#999
```

## Edge 一覧

データソース上の参照方向（`A -> B` = 「A のデータに B への参照がある」）で記載。

### リポジトリ間

| from           | to                  | データソース                          |
| -------------- | ------------------- | ------------------------------------- |
| bioproject     | umbrella-bioproject | BioProject XML / DB `umbrella_info`   |
| bioproject     | hum-id              | BioProject XML `LocalID`              |
| bioproject     | geo                 | BioProject XML `CenterID`             |
| biosample      | bioproject          | BioSample DB/XML + SRA Accessions.tab |
| insdc-assembly | bioproject          | assembly_summary_genbank.txt          |
| insdc-assembly | biosample           | assembly_summary_genbank.txt          |
| insdc-assembly | insdc-master        | assembly_summary_genbank.txt          |
| insdc-master   | bioproject          | assembly_summary + Trad organism list |
| insdc-master   | biosample           | assembly_summary + Trad organism list |
| gea            | bioproject          | GEA IDF `Comment[BioProject]`         |
| gea            | biosample           | GEA SDRF `Comment[BioSample]`         |
| metabobank     | bioproject          | MetaboBank IDF `Comment[BioProject]`  |
| metabobank     | biosample           | MetaboBank SDRF `Comment[BioSample]`  |
| jga-study      | hum-id              | JGA XML `STUDY_ATTRIBUTES`            |
| jga-study      | pubmed-id           | JGA XML `PUBLICATIONS`                |
| trad           | bioproject          | Trad DB `link_pr_ac` + `project`      |
| trad           | biosample           | Trad DB `link_pr_ac` + `project`      |
| trad           | sra-run             | Trad DB `link_pr_ac` + `project`      |
| gea            | sra-experiment      | GEA SDRF `Comment[SRA_EXPERIMENT]`    |
| gea            | sra-run             | GEA SDRF `Comment[SRA_RUN]`           |
| jvar-study     | bioproject          | JVar Excel `BioProject Accession`     |
| jvar-sample    | biosample           | JVar Excel `BioSample Accession`      |

### SRA 内部

全て Accessions.tab 由来。

| from           | to             | 備考                      |
| -------------- | -------------- | ------------------------- |
| sra-submission | sra-study      |                           |
| sra-study      | sra-experiment |                           |
| sra-study      | sra-analysis   |                           |
| sra-submission | sra-analysis   | Study 空の場合の fallback |
| sra-experiment | sra-run        |                           |
| sra-experiment | sra-sample     |                           |
| sra-run        | sra-sample     |                           |
| bioproject     | sra-study      |                           |
| bioproject     | sra-experiment | denormalized              |
| bioproject     | sra-run        | denormalized              |
| bioproject     | sra-analysis   | denormalized              |
| biosample      | sra-sample     |                           |
| biosample      | sra-experiment | denormalized              |
| biosample      | sra-run        | denormalized              |
| biosample      | sra-analysis   | denormalized              |

### JGA 内部

CSV の参照方向（parent → child）で記載。

| parent     | child      | CSV                            |
| ---------- | ---------- | ------------------------------ |
| study      | experiment | experiment-study-relation.csv  |
| study      | analysis   | analysis-study-relation.csv    |
| experiment | data       | data-experiment-relation.csv   |
| experiment | sample     | experiment-sample-relation.csv |
| dataset    | data       | dataset-data-relation.csv      |
| dataset    | analysis   | dataset-analysis-relation.csv  |
| dataset    | policy     | dataset-policy-relation.csv    |
| policy     | dac        | policy-dac-relation.csv        |
| analysis   | sample     | analysis-sample-relation.csv   |
| analysis   | data       | analysis-data-relation.csv     |

### JGA/AGD 申請管理

| from           | to                 | mechanism                            |
| -------------- | ------------------ | ------------------------------------ |
| J-DS           | JGA submission     | `submission_permission` テーブル     |
| J-DU           | jga-dataset (JGAD) | `use_permission` テーブル            |
| JSUB           | JGA submission     | `accession.alias` マッピング         |
| JGA submission | hum-id             | `metadata` XML の `nbdc_number` 属性 |
