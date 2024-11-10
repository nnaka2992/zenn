
---
title: "Cloud SQL for PostgreSQLで全文検索を試す"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["db", "PostgreSQL", "Google Cloud"]
published: true
---

# 検証環境の作成
## CloudSQL 構成
- Cloud SQL Editions
  - Enterprise
- Database Engine
  - PostgreSQL 13.15
- Region
  - asia-northeast1(東京)
- Zone
  - シングルゾーン
- マシンスペック
  - 2 vCPU
  - 3.75GB of RAM
- flag
  - `cloudsql.enable_pg_bigm`
    - True

## PostgreSQLのセットアップ
1. データベースユーザーを作成

postgresユーザーで下記のコマンドを実行する。

```sql
CREATE ROLE test WITH LOGIN ENCRYPTED PASSWORD '********';
GRANT test to postgres;
```

2. データベースを作成

postgresユーザーで下記のコマンドを実行する。

```sql
CREATE DATABASE pgbigm_test WITH OWNER=test;
```

3. pg_bigmを有効化

postgresユーザーで下記のコマンドを実行する。

```sql
CREATE EXTENSION pg_bigm;
```

4. テーブルを作成

testロールで下記のコマンドを実行する。

```sql
CREATE TABLE book_contents (
    id SERIAL NOT NULL,
    title VARCHAR(256) NOT NULL,
    line_num INTEGER NOT NULL,
    sentence TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY(id)
);
```

## 検索対象のデータセットをロード
今回検索の性能確認には青空文庫のテキストのみアーカイブである、[aozorahack/aozorabunko_text](https://github.com/aozorahack/aozorabunko_text)を利用します。

インサートを行うスクリプトの詳細は以下を参照。
利用方法はaozorabunko_textを展開したディレクトリで実行。

@[gist](https://gist.github.com/nnaka2992/eef622f0ab25e7e8e9585251aac13620)

## pg_bigmを利用するテーブルを作成する。

testロールで下記のコマンドを実行する。

```sql
CREATE TABLE book_contents_bigm (
    id SERIAL NOT NULL,
    title VARCHAR(256) NOT NULL,
    line_num INTEGER NOT NULL,
    sentence TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY(id)
);
CREATE INDEX book_contens_bigm_index ON book_contents_bigm USING gin (sentence gin_bigm_ops);
```

```sql
insert into book_contents_bigm
select *
from book_contents;
```

# pg_bigmを利用する

ここまでpg_bigmを利用するテーブルと利用しないテーブルで検証する準備ができたので、実際にクエリや実行計画を出力して比較していきます。

## SELECTの速度を確認する。
シンプルに日本語文字列の格納されている`sentence`カラムに対して`%メロス%`で中間一致検索を試します。

### pg_bigmを利用しないパターン

pg_bigmを利用していない`book_contents`テーブルに対して検索します。
```sql
SELECT *
FROM book_contents
WHERE sentence like '%メロス%';
```
![pg_bigmを有効化していないテーブルで検索する](/images/use_pgbigm_on_cloudsql/select_without_pg_bigm.gif)

### pg_bigmを利用するパターン

pg_bigmを利用していない`book_contents_bigm`テーブルに対して検索します。
```sql
SELECT *
FROM book_contents_bigm
WHERE sentence like '%メロス%';
```
![pg_bigmを有効化したテーブルで検索する](/images/use_pgbigm_on_cloudsql/select_with_pg_bigm.gif)

### SELECT結果(パフォーマンス)の違い
もちろんデータベースですので、キャッシュのデータ状態などによって速度が変わることはありますが、pg_bigmを利用しているテーブルはそうでないテーブルに対して無視できない程度に高速化されていることが分かります。
比較的小規模なデータセットでこれだけの違いが出ているので、より大規模な実サービスのデータベースではよりおおきな差がでることを想定できます。

## 実行計画の比較

上記のSELECTの速度の確認で利用したクエリの実行計画を観察します。

### pg_bigmを利用しないパターン

```sql
explain analyze
select *
from book_contents
where sentence like '%メロス%'
```

```text
Gather (cost=1000.00..111920.66 rows=305 width=216) (actual time=21.243..3140.102 rows=151 loops=1)
Workers Planned: 2
Workers Launched: 2
-> Parallel Seq Scan on book_contents (cost=0.00..110890.16 rows=127 width=216) (actual time=15.952..3128.825 rows=50 loops=3)
Filter: (sentence ~~ '%メロス%'::text)
Rows Removed by Filter: 1025368
Planning Time: 0.578 ms
Execution Time: 3140.242 ms
```

### pg_bigmを利用するパターン

```sql
explain analyze
select *
from book_contents_bigm
where sentence like '%メロス%'
```

```text
Bitmap Heap Scan on book_contents_bigm (cost=46.36..1214.52 rows=304 width=218) (actual time=0.176..0.664 rows=151 loops=1)
Recheck Cond: (sentence ~~ '%メロス%'::text)
Rows Removed by Index Recheck: 2
Heap Blocks: exact=106
-> Bitmap Index Scan on book_contens_bigm_index (cost=0.00..46.28 rows=304 width=0) (actual time=0.156..0.156 rows=153 loops=1)
Index Cond: (sentence ~~ '%メロス%'::text)
Planning Time: 1.209 ms
Execution Time: 0.995 ms
```

### 実行計画の違い



# まとめ

