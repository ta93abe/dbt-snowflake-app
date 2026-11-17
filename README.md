# dbt_snowflake_app

Snowflake 上のアプリケーションオブジェクトを **dbt のモデルとして** 管理するためのカスタムマテリアライゼーション集です。

| materialization | 作成されるオブジェクト | ステータス |
| --- | --- | --- |
| `streamlit` | [Streamlit in Snowflake](https://docs.snowflake.com/en/sql-reference/sql/create-streamlit) アプリ | 開発中 |
| `snowflake_dashboard` | [Snowflake Dashboards (Dashboards 2.0)](https://docs.snowflake.com/en/user-guide/snowflake-dashboards) | 実験的（後述） |

アプリやダッシュボードを dbt モデルとして定義することで、次のことができます。

- `ref()` で依存テーブルを宣言し、**lineage（DAG / dbt docs）にアプリまで含める**
- `dbt build --select +my_app` で「上流モデル → アプリ」を一括デプロイする
- `target` ごと（dev / prod）に database・schema・warehouse を切り替える
- `grants` などの dbt 標準 config で権限を管理する

---

## 要件

- dbt `>= 1.10`（dbt v2 / Fusion engine に対応）
- dbt-snowflake `>= 1.10`（dbt v1 系で使う場合）
- Snowflake ロールに以下の権限があること
  - 対象スキーマへの `CREATE STREAMLIT` / `CREATE STAGE`
  - `QUERY_WAREHOUSE` に指定する warehouse への `USAGE`
  - コンテナランタイムを使う場合は compute pool への `USAGE`

## インストール

利用する側の dbt プロジェクトの `packages.yml` に追加します。

```yaml
packages:
  - git: "https://github.com/ta93abe/dbt-snowflake-app.git"
    revision: 0.1.0 # タグかコミット SHA で固定することを推奨
```

dbt Hub 公開後は次の書き方になります。

```yaml
packages:
  - package: ta93abe/dbt_snowflake_app
    version: [">=0.1.0", "<0.2.0"]
```

```bash
dbt deps
```

パッケージ内のマテリアライゼーションは `dbt deps` だけで利用可能になります。追加の `dispatch` 設定は不要です。

---

## `streamlit` マテリアライゼーション

### 仕組み

1. アプリのソース（`.py`、`environment.yml` / `pyproject.toml` など）をステージへ `PUT` する
2. `CREATE OR REPLACE STREAMLIT ... FROM @stage/... MAIN_FILE = '...'` を実行する
3. `grants` config があれば `GRANT USAGE ON STREAMLIT` を適用する

### 使い方

アプリのソースは dbt プロジェクト内の任意のディレクトリに置きます。

```text
my_dbt_project/
├── dbt_project.yml
├── models/
│   ├── marts/
│   │   └── fct_sales.sql
│   └── apps/
│       └── sales_app.sql        # ← dbt モデル（アプリの定義）
└── streamlit/
    └── sales_app/
        ├── streamlit_app.py
        └── environment.yml
```

`models/apps/sales_app.sql`:

```sql
{{
  config(
    materialized = 'streamlit',
    grants = {'usage': ['ANALYST']},
    meta = {
      'app_dir': 'streamlit/sales_app',   -- dbt プロジェクトルートからの相対パス
      'main_file': 'streamlit_app.py',
      'query_warehouse': 'ANALYTICS_WH',
      'title': 'Sales Dashboard'
    }
  )
}}

-- アプリが参照するモデルを宣言（lineage 用。本文の SQL は実行されない）
-- depends_on: {{ ref('fct_sales') }}
```

アプリ側からは、dbt が解決したテーブル名を直接書くか、後述の `app_context` を使って参照します。

```python
# streamlit/sales_app/streamlit_app.py
import streamlit as st
from snowflake.snowpark.context import get_active_session

session = get_active_session()
df = session.table("ANALYTICS.MARTS.FCT_SALES").to_pandas()
st.dataframe(df)
```

### コンテナランタイム

```sql
{{
  config(
    materialized = 'streamlit',
    meta = {
      'app_dir': 'streamlit/sales_app',
      'main_file': 'streamlit_app.py',
      'query_warehouse': 'ANALYTICS_WH',
      'runtime_name': 'SYSTEM$ST_CONTAINER_RUNTIME_PY3_11',
      'compute_pool': 'STREAMLIT_POOL'
    }
  )
}}
```

コンテナランタイムでは依存パッケージを `pyproject.toml` または `requirements.txt` で指定します。

### 既存ステージ / Git リポジトリから作成する

ソースが既にステージや Snowflake の Git リポジトリにある場合は、`PUT` をスキップして `source_location` を指定します。

```sql
{{
  config(
    materialized = 'streamlit',
    meta = {
      'source_location': '@my_db.my_schema.my_git_repo/branches/main/streamlit/sales_app',
      'main_file': 'streamlit_app.py',
      'query_warehouse': 'ANALYTICS_WH'
    }
  )
}}
```

### config 一覧

dbt v2 では `config` 直下のキーが dbt 公式の設定用に予約されているため、このパッケージ独自の設定はすべて `meta` の中に書きます。`dbt_project.yml` やプロパティ YAML でまとめて指定する場合も同様です。

```yaml
# dbt_project.yml
models:
  my_dbt_project:
    apps:
      +materialized: streamlit
      +meta:
        query_warehouse: ANALYTICS_WH
```

`meta` 内のキー:

| key | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `app_dir` | `source_location` と排他 | - | プロジェクトルートからのソースディレクトリ。配下のファイルをステージへ `PUT` する |
| `source_location` | `app_dir` と排他 | - | 既存のステージ / Git リポジトリのパス |
| `main_file` | ✓ | `streamlit_app.py` | エントリポイントのファイル名 |
| `query_warehouse` | ✓ | - | アプリのクエリを実行する warehouse |
| `title` | | モデル名 | Snowsight 上の表示名 |
| `runtime_name` | | warehouse ランタイム | 例: `SYSTEM$ST_CONTAINER_RUNTIME_PY3_11` |
| `compute_pool` | コンテナ時 ✓ | - | コンテナランタイムで使う compute pool |
| `stage_name` | | `<model_name>_stage` | `app_dir` 使用時にアップロード先として作成する内部ステージ名 |
| `external_access_integrations` | | - | 外部アクセス統合のリスト |

`meta` の外（dbt 標準 config）で使えるもの:

| key | 説明 |
| --- | --- |
| `grants` | 例: `{'usage': ['ANALYST']}` |
| `description` | オブジェクトの `COMMENT` に設定される |

database / schema / alias は通常の dbt モデルと同じく `generate_schema_name` などのルールで決まります。

### 注意点

- `app_dir` による `PUT` は **dbt を実行するマシンのローカルファイル** をアップロードします。dbt Cloud などファイルシステムにアクセスしにくい環境では、Git リポジトリ連携と `source_location` の併用を推奨します。
- モデル本文の SQL は実行されません。`ref()` は lineage のためだけに書きます。
- dbt v2（Fusion engine）での `app_dir`（`PUT file://...`）の動作は未検証です。dbt v2 では `source_location` を第一の選択肢にしてください。
- dbt v2 はカスタムマテリアライゼーションのモデルを静的解析の対象外として扱います。それでも解析エラーになる場合は、モデルに `static_analysis = 'off'` を指定してください。
- `CREATE OR REPLACE` で再作成するため、Snowsight 上で直接加えた変更は次回の `dbt run` で上書きされます。

---

## `snowflake_dashboard` マテリアライゼーション（実験的）

Dashboards 2.0 は Snowsight Workspaces 上で `.dash` ファイルとして作られます。タイルの SQL では `{{ filter('name') }}` 形式でフィルタを参照します。

> [!WARNING]
> 2026-11 時点で、Dashboards 2.0 を作成・更新する公開 DDL（`CREATE DASHBOARD` 相当）は確認できていません。このマテリアライゼーションの実装方法は、Snowflake 側の API / DDL の公開状況に合わせて変わる可能性があります。

想定している使い方:

```text
my_dbt_project/
├── models/apps/sales_dashboard.sql
└── dashboards/
    └── sales.dash
```

```sql
{{
  config(
    materialized = 'snowflake_dashboard',
    meta = {
      'dashboard_file': 'dashboards/sales.dash',
      'query_warehouse': 'ANALYTICS_WH'
    }
  )
}}

-- depends_on: {{ ref('fct_sales') }}
```

`.dash` ファイル内のタイル SQL は Dashboards 2.0 の `{{ filter('...') }}` 構文を含むため、dbt の Jinja としてはレンダリングしません（ファイルとしてそのままアップロードします）。

---

## 開発

```text
.
├── dbt_project.yml
├── macros/
│   ├── materializations/
│   │   ├── streamlit.sql
│   │   └── snowflake_dashboard.sql
│   └── utils/              # PUT・ステージ作成・grants などの共通処理
└── integration_tests/      # このパッケージを読み込む検証用 dbt プロジェクト
    ├── dbt_project.yml
    ├── packages.yml        # local: ../
    ├── models/
    └── streamlit/
```

統合テストの実行:

```bash
cd integration_tests
dbt deps
dbt build --target dev
```

dbt v1 系での非推奨警告の確認（dbt v2 ではエラーになるため、ゼロにしておく）:

```bash
dbt parse --no-partial-parse --show-all-deprecations
```

マテリアライゼーションのマクロでは、独自設定を `config.get('meta')` から読みます（`config.get('app_dir')` のように直接読まない）。

`profiles.yml` はリポジトリに含めません。`~/.dbt/profiles.yml` か環境変数で Snowflake の接続情報を設定してください。

## ライセンス

[Apache License 2.0](LICENSE)

## 参考

- [CREATE STREAMLIT | Snowflake Documentation](https://docs.snowflake.com/en/sql-reference/sql/create-streamlit)
- [Understanding the different types of Streamlit objects](https://docs.snowflake.com/en/developer-guide/streamlit/migrations-and-upgrades/overview)
- [Snowflake Dashboards | Snowflake Documentation](https://docs.snowflake.com/en/user-guide/snowflake-dashboards)
- [dbt: Create new materializations](https://docs.getdbt.com/guides/create-new-materializations)
