---
title: "sqldefとpgrollを利用したPostgreSQLでのスキーマブルーグリーンデプロイメント"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PostgreSQL"]
published: true
published_at: 2023-12-11 08:30
---
この記事はこのエントリー以下のアドベントカレンダーの11日目の記事です。
- [3-shake Advent Calendar 2023](https://qiita.com/advent-calendar/2023/3-shake)
  昨日は[toyb0x](https://zenn.dev/toyb0x)による
  [TODOコメントをチケット管理するためのESLint Custom Rule](https://zenn.dev/toyb0x/articles/55e03a00c75468)でした。
- [PostgreSQL Advent Calendar 2023](https://qiita.com/advent-calendar/2023/postgresql)
  昨日は[@ozozaty](https://twitter.com/onozaty)による
  [PostgreSQLのjsonb型でJSONパス式(JSONPath)を使う](https://zenn.dev/onozaty/articles/postgresql-jsonpath)でした。

## はじめに
PostgreSQLではDDLはその性質からテーブルレベルでロックを取得してしまいます。

SREやPlatform EngineeringなどDevOpsのプラクティスを実践する組織ではリリースが多くなり、必然的にデータベースへのDDL実行も多くなるでしょう。

そのためどんなに極小のDDLによるロックも積れば大きな停止時間になってしまい、ユーザーの負利益が増大します。

MySQLは昔からWeb系システムで採用されることが多かったからか、オンラインDDLへの対応が進んでいます。一方でPostgreSQLではまだまだオンラインDDLには課題が多く、DDLによるテーブルロックを避けるのが難しいです。

この記事では`CREATE TABLE`文とデータベース上のテーブル定義を比較して`ALTER TABLE`を実行する[sqldef](https://k0kubun.hatenablog.com/entry/2018/08/25/114455)と2023年10月にxataというServerless PostgreSQLを提供している企業がOSSとして公開した[^1][pgroll](https://github.com/xataio/pgroll)を組み合わせたオンラインDDLを試します。

## 環境構築
### ツールのインストール
sqldefをインストールする
```
> curl -LO https://github.com/sqldef/sqldef/releases/download/v0.16.13/psqldef_linux_amd64.tar.gz
> tar xf psqldef_linux_amd64.tar.gz
> rm psqldef_linux_amd64.tar.gz
> ./psqldef --version
v0.16.13
```

pgrollをインストールする
```
> curl -LO https://github.com/xataio/pgroll/releases/download/v0.4.1/pgroll.linux.amd64
> mv pgroll.linux.amd64 pgroll
> chmod +x pgroll
> ./pgroll -v
pgroll version 0.4.1
```
### データベースの構築
docker composeを実行してPostgreSQLインスタンスを起動する。
```bash
> ls 
docker-compose.yaml

> cat docker-compose.yaml
version: '3'
services:
  postgres:
    container_name: postgres
    image: postgres:16.1-alpine3.18
    restart: always
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "postgres"

> docker compose up -d
[+] Running 1/1
 ✔ Container postgres  Started                                                                                                     0.0s 
```

## sqldefとは？
sqldefとは[Ridgepole](https://techlife.cookpad.com/entry/2014/08/28/194147)にインスパイアされたデータベーススキーママイグレーションツールです。

`CREATE TABLE`文を書くことでデータベースサーバ上に存在する対象テーブルを比較し、`ALTER TABLE`文を生成しデータベースへのDDL実行までを行なってくれるツールです。

既存のスキーママイグレーションツールに比べて一般的なDDLで表現することができ、`ALTER TABLE`の管理をしなくても良いというメリットがあります。

普通の利用方法ではsqldefを利用してDDLの実行まで行ないますが、今回はオンラインDDLへの対応のため`CREATE TABLE`と実際のテーブルの差分を`ALTER TABLE`として出力する機能(`--dry-run`)のみを利用します。

### 使い方
以下のような`CREATE TABLE`文について考えます。
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    body TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
テーブルが存在しない状態で上記のテーブル定義を利用してsqldefを実行すると以下の様になります。
```bash
> PGPASSWORD=postgres ./psqldef -h localhost -p 5432 -U postgres postgrse --export
-- No table exists --

> PGPASSWORD=postgres ./psqldef -h localhost -p 5432 -U postgres postgrse --dry-run < ddl/posts.sql
-- dry run --
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    body TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
dry runしたDDLに問題がなければ`--dry-run`を外して実行するとDDLを実行することが出来ます。
```bash
> PGPASSWORD=postgres ./psqldef -h localhost -p 5432 -U postgres postgres < ddl/posts.sql
-- Apply --
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    body TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

> PGPASSWORD=postgres psql -h localhost -p 5432 -U postgres postgres
psql (14.9 (Ubuntu 14.9-0ubuntu0.22.04.1), server 16.1)
WARNING: psql major version 14, server major version 16.
         Some psql features might not work.
Type "help" for help.

postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | posts | table | postgres
(1 row)
```
テーブルが作成されたことを確認出来たのでblogテーブルの定義を変更してみます。ブログポストのテーブルなので`author`カラムがあっても良いかもしれません。
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    author VARCHAR(255),
    body TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
この新しいテーブル定義をもとにsqldefをdry runで実行すると以下の様になります。
```bash
> PGPASSWORD=postgres ./psqldef -h localhost -p 5432 -U postgres postgres --dry-run < ddl/posts_new.sql
-- dry run --
ALTER TABLE "public"."posts" ADD COLUMN "author" varchar(255);
```
このようにCREATE TABLE文からALTER TABLE文が生成されます。今回はこれをメインで利用します。

ちなみに`--dry-run`を利用すれば先程と同様DDLが実行されます。
```bash
> PGPASSWORD=postgres ./psqldef -h localhost -p 5432 -U postgres postgres < ddl/posts_new.sql
-- Apply --
ALTER TABLE "public"."posts" ADD COLUMN "author" varchar(255);

> PGPASSWORD=postgres ./psqldef -h localhost -p 5432 -U postgres postgres --export
CREATE TABLE "public"."posts" (
    "id" serial NOT NULL,
    "title" character varying(255),
    "body" text,
    "created_at" timestamp DEFAULT CURRENT_TIMESTAMP,
    "updated_at" timestamp DEFAULT CURRENT_TIMESTAMP,
    "author" character varying(255),
    PRIMARY KEY ("id")
);
```

## pgrollとは？
pgrollとはxataというServerless PostgreSQLを提供している会社が開発したPostgreSQLで柔軟なスキーママイグレーションを実現するツールです。

jsonで記載された独自のオペレーション定義を元にViewやTriggerを駆使してオンラインDDLや2つのバージョンのスキーマを平行稼動させたりすることが出来ます。

### 使い方
pgrollの使用開始時には内部状態を保存するのに必要となる`pgroll`スキーマを作成します。
```bash
> ./pgroll init --postgres-url postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable
 SUCCESS  Initialization complete
```
これでpgrollを利用する準備が出来ました。

次にpgrollでDDLを実行するための定義ファイルを作成します。pgrollでスキーマへのマイグレーションを実行するにはjsonで記述されたオペレーション定義ファイルが必要となります。
```json
{
  "name": "posts_create_table",
  "operations": [
    {
      "create_table": {
        "name": "posts",
        "columns": [
          {
            "name": "id",
            "type": "serial",
            "pk": true
          },
          {
            "name": "title",
            "type": "varchar(255)"
          },
          {
            "name": "body",
            "type": "text"
          },
          {
            "name": "created_at",
            "type": "timestamp",
            "default": "current_timestamp"
          },
          {
            "name": "updated_at",
            "type": "timestamp",
            "default": "current_timestamp"
          }
        ]
      }
    }
  ]
}
```
内容としてはsqldefで実行した`CREATE TABLE`文と同等のものです。

このjsonの構造から見てとれるように実行する操作は複数の操作を一つの単位として扱えるようで、依存関係のあるリソースを一度に作ったり、リソースの削除と代替となるリソースの作成を同じ単位で扱うことが出来るようです。

pgrollではjsonでの独自記法以外にSQLを直接実行する方法もあるようですが、機能が制限されており非推奨なようです。

ここで定義したオペレーションは以下のコマンドで実行することが出来ます。
```bash
> ./pgroll start ddl/posts.json --postgres-url postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable
 SUCCESS  New version of the schema available under the postgres "public_posts" schema
```
`start`はスキーマ変更を開始するコマンドです。このコマンドを実行するとオペレーション定義ファイルに記載したDDLの実行を開始します。

pgrollによるDDL実行は`start`で古いバージョンのスキーマ定義に対する新しいバージョンのスキーマを平行稼動を開始させ、`complete`コマンドで古いバージョンのスキーマを無効化する(=スキーママイグレーションを完了させる)という順序で行なわれます。

この`start`コマンドでは上記の平行稼動を開始させる部分のみを行なったということです。

スキーママイグレーションの実行が完了したので早速テーブルを利用したいところですが、この状態では`relation does not exist`のエラーが発生してしまいます。
```bash
> PGPASSWORD=postgres psql -h localhost -p 5432 -U postgres postgres -c "SELECT count(1) from posts;"
ERROR:  relation "posts" does not exist
LINE 1: SELECT count(1) from posts;
                             ^
```
このエラーはpgrollの動作に起因するものです。pgrollではテーブルを`${schema_name}_${operation_name}`というスキーマに作成します。

つまり今回は`public_create_posts_table`スキーマにテーブルが存在するという訳です。[^2]

作成されたテーブルを利用可能にするには`search_path`を通してあげれば良いです。
```bash
> PGPASSWORD=postgres psql -h localhost -p 5432 -U postgres postgres

postgres=# SET search_path TO 'public_posts_create_table';
SET
postgres=# SELECT COUNT(1) FROM posts;
 count 
-------
     0
(1 row)
```

これで今度こそ`posts`テーブルが利用可能になりました。

新しく作られたテーブルの動作などが問題ないことを確認したら最後に`complete`コマンドを実行します。
```bash
> ./pgroll complete ddl/posts.json --postgres-url postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable
 SUCCESS  Migration successful!

> PGPASSWORD=postgres psql -h localhost -p 5432 -U postgres postgres

postgres=# SELECT count(1) FROM posts;
 count 
-------
     0
(1 row)

```
`complete`コマンドの実行後には`public_create_posts_table`にあったテーブルは`public`スキーマ、つまり利用することを指定していたスキーマに移動されるようです。[^3]

`ALTER TABLE`など既存のテーブルに対する変更を行なうDDLの場合、平行稼動やオンラインDDL相当のことが出来ると嬉しいですが今回は`CREATE TABLE`のため気にすることは少なかったです。そういった場合は`start`コマンドに`--complete`フラグを付けることでDDL発行と同時に`complete`コマンドも実行するという動作を指示することができます。

これで`CREATE TABLE`のオペレーションが完了しました。

次は`ALTER TABLE`を実行します。ALTER TABLEを実行する場合もCREATE TABLEと同様にJSONで記載されたオペレーションを定義したファイルが必要です。
```json
{
  "name": "posts_add_column",
  "operations": [
    {
      "add_column": {
        "table": "posts",
        "column": {
          "name": "author",
          "type": "varchar(255)",
          "nullable": true
        }
      }
    }
  ]
}
```
内容としてはsqldefのdry runで生成した`ALTER TABLE`文と同等のものです。
```
ALTER TABLE ""."posts" ADD COLUMN "author" varchar(255);
```
このjsonファイルを利用して`start`コマンドを実行することで、`ALTER TABLE`を開始することが出来ます。
```bash
> ./pgroll start ddl/posts.json --postgres-url postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable
 SUCCESS  New version of the schema available under the postgres "public_posts_new" schema

> PGPASSWORD=postgres psql -h localhost -p 5432 -U postgres postgres
postgres=# \dn
                List of schemas
           Name            |       Owner       
---------------------------+-------------------
 pgroll                    | postgres
 public                    | pg_database_owner
 public_posts_add_column   | postgres
 public_create_posts_table | postgres
(4 rows)

postgres=# \d posts
                                                  Table "public.posts"
       Column       |            Type             | Collation | Nullable |                    Default                    
--------------------+-----------------------------+-----------+----------+-----------------------------------------------
 id                 | integer                     |           | not null | nextval('_pgroll_new_posts_id_seq'::regclass)
 title              | character varying(255)      |           | not null | 
 body               | text                        |           | not null | 
 created_at         | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 updated_at         | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 _pgroll_new_author | character varying(255)      |           |          | 
Indexes:
    "_pgroll_new_posts_pkey" PRIMARY KEY, btree (id)

```
さてこの`posts`の定義をみると新しいカラム名に`_pgroll_new_`というプレフィックスが付いていることが気になると思います。その理由はこの時点では`public.posts`がそのまま利用される前提では無いからです。

この時点では以下のように`public_create_posts_table.posts`Viewと`public_posts_add_column.posts`Viewが存在しており、古いアプリケーションと新しいアプリケーションからそれぞれが使われることを想定しているためです。

```bash
postgres=# \d public_posts_create_table.posts 
                  View "public_posts_create_table.posts"
   Column   |            Type             | Collation | Nullable | Default 
------------+-----------------------------+-----------+----------+---------
 body       | text                        |           |          | 
 created_at | timestamp without time zone |           |          | 
 updated_at | timestamp without time zone |           |          | 
 id         | integer                     |           |          | 
 title      | character varying(255)      |           |          | 

postgres=# \d public_posts_add_column.posts 
                   View "public_posts_add_column.posts"
   Column   |            Type             | Collation | Nullable | Default 
------------+-----------------------------+-----------+----------+---------
 id         | integer                     |           |          | 
 body       | text                        |           |          | 
 title      | character varying(255)      |           |          | 
 created_at | timestamp without time zone |           |          | 
 updated_at | timestamp without time zone |           |          | 
 author     | character varying(255)      |           |          | 

```
つまりこの時点では論理的に2つのテーブルのバージョンが存在しているということです。

実際にそれぞれのバージョンが動いている想定でクエリを実行してみます。

SQLは以下のようなスクリプト経由で実行します。内容としては簡単で引数として`new`か`old`の何れかと実行時間(sec)をとり、指定されたバージョンに対応するINSERT文を指定された時間だけ実行するというものです。

全てのINSERTは別のセッションで行なわれます。
```bash
#!/bin/env bash

lorem() { var=$1; perl -E "use Text::Lorem;my \$text = Text::Lorem->new();\$paragraphs = \$text->paragraphs($1);say \$paragraphs;" -- -var=$var; }

if [ $# -ne 2 ] || [ "$1" != "old" -a "$1" != "new" ]; then
    echo "Usage: $0 <old|new> <second>"
    exit 1
else
    VERSION="$1"
    if [ "${VERSION}" == "old" ]; then
        QUERY="SET search_path TO $1; INSERT INTO posts (title, body) VALUES ('$(lorem 1 | head -c 254)','$(lorem 10)');"
    else
        QUERY="SET search_path TO $1; INSERT INTO posts (title, body, author) VALUES ('$(lorem 1 | head -c 254)','$(lorem 10)','$(lorem 1 | head -c 10)');"
    fi
    second="$2"
fi

start=$(date +%s)
end=$((start + second))
while true; do
    psql -h localhost -p 5432 -U postgres postgres -c """${QUERY}""" > /dev/null
    if [[ $(date +%s) -ge $end ]]; then
        break
    fi
    echo "Finish ${VERSION}"
done
```
以下のようにコマンドを実行すると、古い状態のテーブルと新しい状態のテーブルが平行して稼動しているように動作していることが分かります。
```
> ./insert old 120 &
> ./insert new 120 &
> psql -h localhost -p 5432 -U postgres postgres
postgres=# select count(1) from posts;
 count 
-------
   315
(1 row)

postgres=# select count(1) from posts;
 count 
-------
   375
(1 row)
postgres=# exit

Finish old
Finish new
```
このようにアプリケーションを動作させて問題なければ`complete`コマンドを実行し、想定外の動作やエラーが多発したときには`rollback`コマンドでスキーマ変更を巻き戻すことも出来ます。

先程`complete`コマンドは実行済みなため、折角なので`rollback`コマンドをためしてみましょう。
```
> ./pgroll rollback ddl/posts_new.json --postgres-url postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable
 SUCCESS  Migration rolled back. Changes made since the last version have been reverted 

> psql -h localhost -p 5432 -U postgres postgres
postgres=# \dn
                List of schemas
           Name            |       Owner       
---------------------------+-------------------
 pgroll                    | postgres
 public                    | pg_database_owner
 public_posts_create_table | postgres
(3 rows)

postgres=# \d posts
                                              Table "public.posts"
   Column   |            Type             | Collation | Nullable |                    Default                    
------------+-----------------------------+-----------+----------+-----------------------------------------------
 id         | integer                     |           | not null | nextval('_pgroll_new_posts_id_seq'::regclass)
 title      | character varying(255)      |           | not null | 
 body       | text                        |           | not null | 
 created_at | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 updated_at | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
Indexes:
    "_pgroll_new_posts_pkey" PRIMARY KEY, btree (id)
```
`start`コマンドで作成された`public_posts_add_column`スキーマは削除され、`public.posts`にあった`_pgroll_new_author`カラムも消えています。

また`rollback`コマンドを実行しても、新しいスキーマで`INSERT`されたデータが消えるようなこともありません。[^4]
```
postgres=# --rollback実行前
postgres=# select count(1) from posts;
 count 
-------
  6816
(1 row)

postgres=# --rollback実行後
postgres=# select count(1) from posts;
 count 
-------
  6816
(1 row)
```

`rollback`コマンドの挙動も確認できたので`start`コマンドを`complete`フラグありで実行して見ましょう。
```
> ./pgroll start ddl/posts_new.json --postgres-url postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable --complete
 SUCCESS  New version of the schema available under the postgres "public_posts_new" schema

> psql -h localhost -p 5432 -U postgres postgres
postgres=# \dn
               List of schemas
          Name           |       Owner       
-------------------------+-------------------
 pgroll                  | postgres
 public                  | pg_database_owner
 public_posts_add_column | postgres
(3 rows)

postgres=# \d posts
                                              Table "public.posts"
   Column   |            Type             | Collation | Nullable |                    Default                    
------------+-----------------------------+-----------+----------+-----------------------------------------------
 id         | integer                     |           | not null | nextval('_pgroll_new_posts_id_seq'::regclass)
 title      | character varying(255)      |           | not null | 
 body       | text                        |           | not null | 
 created_at | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 updated_at | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 author     | character varying(255)      |           |          | 
Indexes:
    "_pgroll_new_posts_pkey" PRIMARY KEY, btree (id)

postgres=# SELECT COUNT(1) FROM posts;
 count 
-------
  6816
(1 row)

postgres=# SELECT count(1) FROM posts WHERE author IS NOT NULL;
 count 
-------
     0
(1 row)
```
今度は`public_posts_add_column`ではなく`public_posts_create_table`が削除されており、想定通り`author`カラムも作成されています。また`public.posts`テーブルにあった6816列は変らず存在し、`author`カラムは全て`null`で入っています。

`public_posts`スキーマでテーブルを作成したと言いながら違うスキーマにテーブルが存在するなど挙動がコマンドの出力と異なったり、DDLを独自記法しか受け付けなかったりと特殊な部分はあれどおもしろい使い方が出来そうなツールです。

## pgrollとsqldefによるオンラインスキーマ変更とブルーグリーンデプロイメントについての考察
ここまでsqldefとpgrollを見てきましたが、この2つのスキーママイグレーションツールはそれぞれ異なるゴールを達成することが目的に見えます。

- sqldef
  `CREATE TABLE`のみを入力としてスキーマの管理を効率的に行なうこと。
- pgroll
  独自記法で拡張性の高い構文を利用して、PostgreSQLのみでは実現できない柔軟なスキーママイグレーションを行なうこと。


はじめにに記載したようにPostgreSQLでは殆どのDDLが`ACCESS EXCLUSIVE MODE`のロックを取得するため、オンラインDDLの実行が課題となります。またPostgreSQLでは複数のスキーマバージョンを持つことが出来ないため、スキーマ変更を共なうアプリケーションのアップデートは人間が注意深く監視し、問題があれば泥臭く修復を行う必要があります。

ここではこの2つのツールを組み合わせることで、データベースエンジニアが慣れしたしんだSQLからデータベーススキーマのブルーグリーンデプロイメントする方法について考えます。

pgroll記法を最初から書けば余計なことを考える必要はありませんが、それでは面白くないのでsqldefを使います。何より`CREATE TABLE`で管理したいので。[^5]

今回考案するフローは以下のようなものです。

1. 開発者がコードリポジトリにコードをプッシュする
2. CI/CDツールがコードのプッシュを起点としてアプリケーションなどのビルドを開始する
3. sqldefでプッシュされたコードのDDLを生成する
4. 3で生成したDDLをpgroll記法に変換する
5. pgrollでスキーママイグレーションを実行する
6. 新しいスキーマを利用するアプリケーションを一部デプロイする
7. 新しいスキーマでエラーレートなどが問題ないことを確認したのち、全てのアプリケーションを新しいバージョンにロールアウトする
8. pgrollでスキーママイグレーションを完了させる
9. 開発者にデプロイメントが完了したことを通知する

![pgroll drawio](https://gist.github.com/assets/66209003/56e7bc5c-cb0c-4fe9-ad09-7f0023e637b8)
[^6]

このフローでは4のDDLをpgroll独自の記法に変換する部分のみ課題として残っています。しかしsqldefがSQLをパースして新たなSQLを生成していたり、pgroll自体が独自記法のjsonからDDLを生成していたりと先行事例には事欠きません。

時間が足らず今回は実物の作成まで至っていませんが、[pg_query_go](https://github.com/pganalyze/pg_query_go)などSQLパーサを利用すれば十分実現可能でしょう。

また今時点でもSQLからCI/CDを開始するという部分を捨てて、pgroll記法のファイルであれば今すぐにでもCI/CDに取り込むことが可能でしょう。

## まとめ
この記事では異なるスキーママイグレーションツールであるsqldefとpgrollを紹介しました。またその2つを組合せたアプリケーションのブルーグリーンデプロイメントに適合したオンラインスキーマ変更について考えました。

相変わらず見積りが甘く、SQLからpgroll記法を生成するところまで手がまわりませんでしたが時間を見付けてやります。

[^1]: [Introducing pgroll: zero-downtime, reversible, schema migrations for Postgres](https://xata.io/blog/pgroll-schema-migrations-postgres)

[^2]: `pgroll start`の実行結果に表示されているものは間違いです。

[^3]: 実際の挙動は少しことなるようですが、この資料では割愛します。詳しくは[How pgroll works under the hood](https://xata.io/blog/pgroll-internals)に記載されています。

[^4]: `INSERT`を行なったのは`VIEW`に対してなのであたりまえと言われればあたりまえです。

[^5]: 個人的な経験ですがアプリケーションエンジニアの声を聞いていると既存のテーブルへの変更を`ALTER TABLE`などで管理することは避けたいというケースが多くありました。また実際そのようなニーズに答えるツールも複数あります。

[^6]: 普段お仕事ではUMLを書く機会が少ないのでゆるふわですが見逃してください。
