---
title: "BigQueryに溜まった使われてないテーブル・ビューを大掃除する🧹" # 記事のタイトル
emoji: "🫧" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: # タグ。["markdown", "rust", "aws"]のように指定する
  - "BigQuery"
  - "GCP"
  - "dbt"
published: true # 公開設定（falseにすると下書き）
published_at: "2023-12-25 23:59" # 公開設定日時（JST）
---

**メリークリスマス🎄🎁**
街は浮かれ気分かもしれませんが、もうすぐ年の瀬です🔔やり残したことはないでしょうか？

....

.......

本当にないですか? いやいや、、、
この記事を見ているならきっとあるはずです。そうです例のアレです...

....


.......


.............

![](https://storage.googleapis.com/zenn-user-upload/4997508c4cc2-20231224.png)

そうです。**大掃除**です🧹🧹🧹
本記事では、BigQuery利用者が作りっぱなしで溜まりに溜まったテーブルを大掃除する方法について記載します。

なお、本記事は [dbt Advent Calendar 2023 最終日の記事](https://qiita.com/advent-calendar/2023/dbt) となります🦌
先にネタバラシをするとdbtは利用しません。
ですが、dbt移行以前のテーブルが残っている方や、dbtで管理できていないテーブルに悩まされている方にも役立つ内容になれば幸いです。

:::message
BigQueryの利用者・野良ダッシュボードが多いと、大掃除による影響が出る可能性があります。
その年の穢れはその年になくしたいところですが、年末年始で出退勤者が不規則になりがちなので慎重にお掃除いただければと思います。
:::

# 全体の方針
テーブルおよびビューの最終更新日および、直近の参照回数を集計し、お掃除して問題ないと判断したものを削除します。

# テーブル・VIEWの最終更新日の取得

下記のクエリで、任意のデータセットにあるテーブル・VIEWの最終更新日時を取得できます。
なお今回は全データセットを対象にしたいため、Google Colaboratoryを利用して、for文でぶん回すことにします。

```sql:任意のデータセットにあるテーブル・VIEWの最終更新日時を取得するクエリ.sql
select
  project_id
  , dataset_id
  , table_id
  , row_count
  , size_bytes
  , round(safe_divide(size_bytes, (1000*1000)),1) as size_mb
  , round(safe_divide(size_bytes, (1000*1000*1000)),2) as size_gb
  , case
    when type = 1 then 'BASE TABLE'
    when type = 2 then 'VIEW'
    when type = 3 then 'EXTERNAL'
    when type = 4 then 'OTHER'
  end type
  , timestamp_millis(creation_time) as creation_time
  , timestamp_millis(last_modified_time) as last_modified_time
from
  `任意のプロジェクト名.任意のデータセット名.__TABLES__`
```

- 全データセットのテーブル・VIEWの最終更新日時を取得するためのコード↓
:::details 全データセットのテーブル・VIEWの最終更新日時を取得するためのコード（クリックでコードを表示する）
```py:全データセットのテーブル・VIEWの最終更新日時を取得するためのコード.py
from google.colab import auth
auth.authenticate_user()

from google.cloud import bigquery
from google.colab import drive
drive.mount('/content/drive')

import pandas as pd
import datetime

# 自身のGCPProjectIDを指定
project_id = '任意のプロジェクトID'
client = bigquery.Client(project=project_id)

# 実行するクエリ
query =  """
            select
              distinct schema_name
            from
              `任意のプロジェクトID.INFORMATION_SCHEMA.SCHEMATA`;
         """

# データフレームで結果を受け取る
df_dataset_name = client.query(query).to_dataframe()

def query_table_info(project_id, dataset_name):
  return f"""
    select
      project_id
      , dataset_id
      , table_id
      , row_count
      , size_bytes
      , round(safe_divide(size_bytes, (1000*1000)),1) as size_mb
      , round(safe_divide(size_bytes, (1000*1000*1000)),2) as size_gb
      , case
        when type = 1 then 'BASE TABLE'
        when type = 2 then 'VIEW'
        when type = 3 then 'EXTERNAL'
        when type = 4 then 'OTHER'
      end type
      , timestamp_millis(creation_time) as creation_time
      , timestamp_millis(last_modified_time) as last_modified_time
    from
      `{project_id}.{dataset_name}.__TABLES__`
  """

# クエリ結果を格納するテーブルの用意
df_table_info = pd.DataFrame(columns=['project_id', 'dataset_id', 'table_id', 'row_count', 'size_bytes', 'size_mb', 'size_gb', 'table_type', 'created_at', 'last_modified_at'])
for i in range(0, len(df_dataset_name)):
  dataset_name = df_dataset_name.schema_name[i]
  query = query_table_info(project_id, dataset_name)
  df_table_info = pd.concat([df_table_info, pd.io.gbq.read_gbq(query, project_id=project_id)]) #クエリ結果を追加していく

# 結果をcsvとして保存する
DATE = datetime.date.today().strftime('%Y_%m_%d')
df_all.to_csv('drive/Shareddrives/任意のファイル名_' + DATE +'.csv', index = False)
```
:::

# テーブルの参照された回数の取得

VIEWの参照された回数は取得できませんが、下記のクエリでテーブルの参照された回数を取得することができます。

```sql:テーブルの参照された回数を取得するクエリ.sql

with jobs_by_project_base as (
  select
    creation_time
    , job_id
    , reservation_id -- BigQuery Editionsを有効にしている場合 not null となる
    , referenced_tables.project_id
    , referenced_tables.dataset_id
    , referenced_tables.table_id
    , query
    , user_email
    , job_type -- # ジョブのタイプ ex/ "QUERY"
    , statement_type -- # クエリステートメントのタイプ ex/"SELECT"
    , state -- # ジョブの実行状態 ex/ "DONE"
    , bi_engine_statistics.bi_engine_mode -- # ex/ "DISABLED", "PARTIAL", "FULL"
    -- 便宜的に配列の0番目の要素のみを取得
    , bi_engine_statistics.bi_engine_reasons[safe_offset(0)].code as bi_engine_reasons_code
    , bi_engine_statistics.bi_engine_reasons[safe_offset(0)].message as bi_engine_reasons_message
    , bi_engine_statistics.acceleration_mode as bi_engine_acceleration_mode
  from
    `任意のプロジェクトID.region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT`, unnest(referenced_tables) as referenced_tables
)

select
  project_id
  , dataset_id
  , table_id
  , count(distinct job_id) AS n_job
from jobs_by_project_base
where
  creation_time between "2023-01-01" and "2023-12-31"
  and state = 'DONE'
group by 1,2,3
```

# テーブルの削除方針

ここまでで取得した、テーブルの最終更新日時とテーブルを参照された回数を参考にして、お掃除対象のテーブルを洗い出して準備します。
お掃除対象のテーブルが洗い出せたら、Google Colaboratoryのfor文で、テーブルのリネーム設定および任意の有効期限の設定のsqlクエリをぶん回せるようにcsvファイルを用意します。

テーブル削除の方針としては、monthly実行が失敗した場合の影響も考慮し、テーブルのリネーム設定および任意の有効期限を設定することとします。
なお、BigQueryではデフォルトで7日間データの復元ができる[タイムトラベル機能(公式ドキュメント)](https://cloud.google.com/bigquery/docs/time-travel?hl=ja)がありますが、今回はタイムトラベルに頼らないで済むような方針としています。

下記のコードのように、用意したcsvファイルを読み込んで、テーブルのリネーム・有効期限の設定を行います。
有効期限が来たら、お掃除対象の物理テーブルが綺麗になくなります🫧

:::details お掃除対象のテーブルに対してリネーム・有効期限の設定を行うためのコード（クリックでコードを表示する）
```py:お掃除対象のテーブルに対してリネーム・有効期限の設定を行う.py
# お掃除対象とするプロジェクト名, データセット名, テーブル名の組み合わせをcsvファイルから読み込む
df_tmp = pd.read_csv('drive/Shareddrives/任意のファイル名.csv')

# リネーム用のクエリの定義
def rename_table_name(PROJECT_ID, DATASET_NAME, TABLE_NAME):
  return f"""
    alter table `{PROJECT_ID}.{DATASET_NAME}.{TABLE_NAME}`
    rename to {TABLE_NAME}_depreacted;

    alter table `{PROJECT_ID}.{DATASET_NAME}.{TABLE_NAME}_depreacted`
    -- -- 月跨ぎを考慮できるように半月分の日数を設定する
    set options (
      expiration_timestamp = timestamp_add(current_timestamp(), interval 17 day) -- 任意の日付を設定する
      );
  """

# for文でクエリのリネーム・有効期限の更新を実行する
project_id = '任意のプロジェクトID'
for i in df_tmp.index:
  dataset_name = df_tmp.table_schema[i]
  table_name = df_tmp.table_name[i]
  rename_query = rename_table_name(project_id, dataset_name, table_name)
  try:
    # クエリ実行のリクエストが走る(結果(データ)を返すクエリではないためエラーとなるので、try文でfor文が途中で止まらないようにする)
    # TODO: クエリ実行のみを行うものに修正できるとよい
    pd.io.gbq.read_gbq(rename_query, project_id=project_id)
  except:
    pass
  print(project_id + "." + dataset_name + "." + table_name + " was renamed!")
```
:::

# VIEWの削除方針
方針としては、ロールバックが行えるようにVIEWクエリのダウンロードを行った上で、VIEWは Cloud Shellでコマンド入力して削除してしまうこととします。

- VIEWクエリダウンロードのためのコマンド
```shell
bq show --format=prettyjson 任意のプロジェクト:データセット名.ビュー名 > ~/Desktop/任意のファイル名.json
```
![](https://storage.googleapis.com/zenn-user-upload/18eaf64a284a-20231226.png)*L列のコマンドを実行すれば一気にクエリをダウンロードできる*

- VIEW削除のためのコマンド
```shell
bq rm -f -t 任意のプロジェクト:データセット名.ビュー名
```
-f オプション(--force)は、確認メッセージを表示せずにリソースを削除するためのコマンドです。
-t オプション(--table)は、テーブルまたはビューを削除するために必須のコマンドです。

ちなみに下記のSQLコマンドでもVIEWを削除できます。
```sql
DROP VIEW 任意のプロジェクト.データセット名.ビュー名;
```

# データセットの削除方針
テーブルおよびVIEWテーブルのお掃除が済み、データセット内に何もなくなったことを確認した後に、Cloud Shellでコマンド入力して削除する方針とします。

- データセット内のテーブル・VIEWの数を数えるコード↓
:::details データセット内のテーブル・VIEWの数を数えるためのコード（クリックでコードを表示する）
```py:データセット内のテーブル・VIEWの数を数えるためのコード.py
from google.colab import auth
auth.authenticate_user()

from google.cloud import bigquery
from google.colab import drive
drive.mount('/content/drive')

import pandas as pd
import datetime

# 自身のGCPProjectIDを指定
project_id = '任意のプロジェクト名'
client = bigquery.Client(project=project_id)

# 実行するクエリ
query_get_schema_name =  """
              select distinct schema_name
              from `任意のプロジェクト名.region-us.INFORMATION_SCHEMA.SCHEMATA` ;
         """

# メソッドに`to_dataframe()`をつけるとpd.DataFrameで結果を受け取れる
df_get_schema_name = client.query(query_get_schema_name).to_dataframe()

def query_number_of_table(project_id, dataset_name):
  return f'''
    with base as (
      select
        '{project_id}' as project_id
        , '{dataset_name}' as dataset_id
    )

    select
      project_id
      , dataset_id
      , case
        when type = 1 then 'BASE TABLE'
        when type = 2 then 'VIEW'
        when type = 3 then 'EXTERNAL'
        when type = 4 then 'OTHER'
      end table_type
      , count(distinct table_id) as n_table
    from
      `{project_id}.{dataset_name}.__TABLES__`
    group by 1,2,3
  '''

df_number_of_table = pd.DataFrame(columns=['project_id', 'dataset_id', 'table_type', 'n_table'])
for i in range(0, len(df_get_schema_name)):
  dataset_name = df_get_schema_name.schema_name[i]
  query = query_number_of_table(project_id, dataset_name)
  df_number_of_table = pd.concat([df_number_of_table, pd.io.gbq.read_gbq(query, project_id=project_id)])

# 結果をcsvとして保存する
DATE = datetime.date.today().strftime('%Y_%m_%d')
df_number_of_table.to_csv('drive/Shareddrives/任意のファイル名_' + DATE +'.csv', index = False)
```
:::

- データセット削除のためのコマンド
VIEWの削除と同様の方法でデータセットを削除します。

:::message alert
誤って必要なデータセットを削除しないようにご注意ください。
テーブルのみ、[タイムトラベル機能(公式ドキュメント)](https://cloud.google.com/bigquery/docs/time-travel?hl=ja)で復元できる可能性がありますが、ビュー、マテリアライズド ビュー、ルーティンなど、データセットに関連付けられている他のオブジェクトは手動で再作成する必要があります。
:::

```shell
bq rm -r -f -d 任意のプロジェクトID:任意のデータセット名
```
-r オプション(--recursive)は、データセットおよびその中のすべてのテーブル、テーブルデータ、モデルを削除するためのコマンドです。
-f オプション(--force)は、確認メッセージを表示せずにリソースを削除するためのコマンドです。
-t オプション(--table)は、テーブルまたはビューを削除するために必須のコマンドです。

ちなみにSQL文でもデータセットの削除は可能です。
```sql
-- 空のデータセットを削除する場合に有効
DROP SCHEMA IF EXISTS 任意のデータセット名;

-- ータセットとそのすべてのコンテンツを削除するには、CASCADE キーワードを使用する
DROP SCHEMA IF EXISTS 任意のデータセット名 CASCADE;
```

## その他Tips
先述のように、テーブルは有効期限を設定することで自動でお掃除が完了する仕組みを作ることができます。
ただし個別のテーブルに有効期限を設定するのは骨が折れるうえ、誤って利用頻度が高い重要なテーブルを削除してしまう可能性もあるかもしれません。

そこで、Tipsとしてデフォルトの有効期限を設定したsandboxデータセット環境を作成することをおすすめします！
sandbox環境を個人あるいはチームごとに作成して利用してもらうことで、手軽にデータをあれこれいじって自動でお掃除がされる環境を運用することができます。
（具体的な方法は、[「データセット プロパティの更新  |  BigQuery  |  Google Cloud」](https://cloud.google.com/bigquery/docs/updating-datasets?hl=ja)をご参照ください。）
![](https://storage.googleapis.com/zenn-user-upload/3b7d04a32523-20231226.png)

# 所感
今回はPythonで手軽にfor文を回したかったので Google Colaboratoryを利用しましたが、やろうと思えば dbt jinja で実装することもできそうです。
（都度、csvを読み込んだりも必要そうなのでdbtで実装するメリットはあまりなさそうですが...）

現実世界のお部屋掃除と異なって、BigQueryのテーブルのお掃除は日々コツコツやるよりもまとめてやった方が良さそうと感じました。
とはいえ、似たようなテーブル名ばかりで混同が起きたり、ストレージコストがかかったりといったデメリットもあるので気になったテーブルは都度お掃除するのが良いと思います。

また、その他Tipsで触れたsandbox環境は個人的にも重宝しているのでよかったら試してみてください！

それではみなさん、綺麗なBigQuery環境で良いお年をお過ごしください〜⛩

# 参考記事
- [公式ドキュメントにない方法で、BigQueryにあるデータセット内の全てのテーブル・ビューの情報を取得してみた - データサイエンス＆サイバーセキュリティ備忘録](https://a7xche.hatenablog.com/entry/2020/09/12/222045)
- [タイムトラベルとフェイルセーフによるデータの保持  |  BigQuery  |  Google Cloud](https://cloud.google.com/bigquery/docs/time-travel?hl=ja)
- [データセット プロパティの更新  |  BigQuery  |  Google Cloud](https://cloud.google.com/bigquery/docs/updating-datasets?hl=ja)をご参照ください。）
- [ビューの管理  |  BigQuery  |  Google Cloud](https://cloud.google.com/bigquery/docs/managing-views?hl=ja)
- [データセットを管理する  |  BigQuery  |  Google Cloud](https://cloud.google.com/bigquery/docs/managing-datasets?hl=ja)
- [bq コマンドライン ツール リファレンス  |  BigQuery  |  Google Cloud](https://cloud.google.com/bigquery/docs/reference/bq-cli-reference?hl=ja)