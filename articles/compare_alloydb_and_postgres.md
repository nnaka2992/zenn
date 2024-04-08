---
title: "AlloyDBの性能を眺める"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [""]
published: false
---
# 概要
Google Cloudが提供するPostgreSQL互換データベースであるAlloyDBのパフォーマンスをトランザクション用途・分析用途の双方から検証する。
今回の検証ではAlloyDBの上限を見定めるのではなく、CloudSQLと比べてどのようなパフォーマンスになるのかに主眼を置く。

# 検証方法
同一VPC上にプロビジョニングしたAlloyDBとCloudSQL for PostgreSQLに対して、Compute Engine上のHammerDBからTPC-C LikeなベンチマークとTPC-H Likeなベンチマークを実行する。
Compute Engineとデータベース間の接続は、Service Network Connection経由のPrivate IPで疎通する。

# 検証環境
## AlloyDB
- クラスタ種類
  - 基本
- PostgreSQL Version
  - PostgreSQL 15に対応
- Region
  - asia-northeast1(東京)
- Zone
  - 選択不可
- マシン
  - 8 vCPU
  - 64 GB RAM
## CloudSQL for PostgreSQL
- PostgreSQL Version
  - PostgreSQL 15.5 (2024/04/08現在最新)
- Edition
  - Enterprise Plus
- マシン
  - 8 vCPU
  - 64 GB RAM
- Region
  - asia-northeast1(東京)
- Zone
  - asia-northeast1-a
- データキャッシュ
  - 有効 (375GB)
- Query Insights
  - 有効 (AlloyDBが自動で有効になるため)
## Compute Engine
- インスタンスタイプ
  - c2-standard-8
    - 8 vCPU
    - 32 GB RAM
- イメージ
  - debian-11-bullseye-v20240312

# 検証
## HammerDBのインストール
HammberDBの実行に必要なlibpqをビルドする。^[APTで取得できる
libpq/postgresql0clientはバージョンが古いため]
```bash
# 依存パッケージをインストール
sudo apt update
sudo apt build-dep postgersql
sudo apt install zlib1g-dev

# PostgreSQLクライアントをビルド
curl -O https://ftp.postgresql.org/pub/source/v15.6/postgresql-15.6.tar.gz
tar xf postgresql-15.6.tar.gz
cd postgresql-15.6
./configure
make -j 8

# PostgreSQLクライアントをインストールする
sudo make install

# 共有ライブラリのパスを通す
export LD_LIBRARY_PATH=/usr/local/pgsql/lib
```

libpqが存在することを確認する。
```bash
sudo find / -name libpq.so
```
:::message
/home/nnaka2992/postgresql-15.6/src/interfaces/libpq/libpq.so
/usr/local/pgsql/lib/libpq.so
:::

HammberDBをインストールする。
```
# HammberDBをダウンロードし解凍
curl -LO https://github.com/TPC-Council/HammerDB/releases/download/v4.10/HammerDB-4.10-Linux.tar.gz
tar xf HammerDB-4.10-Linux.tar.gz
```
HammberDBからPostgreSQLに対してベンチマークが実行可能なことを確認する。
```bash
# HammberDBを起動
cd HammberDB-4.10
./hammerdbcli

```
```hammerdbcli
HammerDB CLI v4.10
Copyright (C) 2003-2024 Steve Shaw
Type "help" for a list of commands
Initialized Jobs on-disk database /tmp/hammer.DB using existing tables (45,056 KB)

hammerdb>librarycheck
...
Checking database library for PostgreSQL
Success ... loaded library Pgtcl for PostgreSQL
...
```

## AlloyDBの性能検証
### TPC-C Like
HammerDBのベンチマーク実行のための情報を設定する。
```hammerdbcli
# DBにPostgreSQLを指定
hammerdb>dbset db pg
# ベンチマークシナリオにTPROC-C(TPC-C Like)を利用することを指定
hammerdb>dbset bm TPROC-C
# PostgreSQLの接続情報を設定
hammerdb>diset connection pg_host xxx.xxx.xxx.xxx
hammerdb>diset connection pg_port 5432
hammerdb>diset tpcc pg_superuser postgres
hammerdb>diset tpcc pg_superuserpass xxxxxx
hammerdb>diset tpcc pg_defaultdbase postgres
hammerdb>diset tpcc pg_user postgres
hammerdb>diset tpcc pg_pass xxxxxx
hammerdb>diset tpcc pg_dbase tpcc
# スケーリングファクターを200に設定 (約10GB)
hammerdb>diset tpcc pg_count_ware 200
# 並列実行数を指定(8並列)
hammerdb>diset tpcc pg_num_vu 8
# ロードしたデータ全てを対象にベンチマークを取得するように設定
hammerdb>diset tpcc pg_allwarehouse true
# ベンチマークの最初の3分は無視するように設定
hammerdb>diset tpcc pg_rampup 3
```
データをロードする。^[10GBほどデータを積み込むので結構時間がかかります。]
```hammerdbcli
hammerdb>buildschema
```
ベンチマークを実行するユーザーと、ログを設定する。
```hammerdbcli
hammerdb>vudestroy
hammerdb>loadscript
hammerdb>vuset vu 8
hammerdb>vuset logtotemp 1
hammerdb>vuset showoutput 1
hammerdb>vuset unique 1
hammerdb>vuset timestamps 1
hammerdb>print vuconf
Virtual Users = 8
User Delay(ms) = 500
Repeat Delay(ms) = 500
Iterations = 1
Show Output = 1
Log Output = 1
Unique Log Name = 1
No Log Buffer = 0
Log Timestamps = 1
```
ベンチマーク実行ユーザーを作成する。
```hammerdbcli
hammerdb>vucreate
```
ベンチマークを実行する。
```hammerdbcli
hammerdb>vurun
...
Vuser 1:TEST RESULT : System achieved 18664 NOPM from 43386 PostgreSQL TPM
Vuser 1:FINISHED SUCCESS
Vuser 4:FINISHED SUCCESS
Vuser 2:FINISHED SUCCESS
Vuser 3:FINISHED SUCCESS
Vuser 5:FINISHED SUCCESS
Vuser 6:FINISHED SUCCESS
Vuser 8:FINISHED SUCCESS
Vuser 9:FINISHED SUCCESS
Vuser 7:FINISHED SUCCESS
ALL VIRTUAL USERS COMPLETE
Benchmark Run jobid=6612DCDE615803E293338333
```
#### 結果
今回のベンチマークでは新規発注18,664件を実行し、43,386トランザクションを実行したようです。
また10GBほど積み込まれたデータは20GBほどに増えていました。

### TPC-H Like
```hammerdbcli
# DBにPostgreSQLを指定
hammerdb>dbset db pg
# ベンチマークシナリオにTPROC-H(TPC-H Like)を利用することを指定
hammerdb>dbset bm TPROC-H
# PostgreSQLの接続情報を設定
hammerdb>diset connection pg_host xxx.xxx.xxx.xxx
hammerdb>diset tpch pg_tpch_superuser postgres
hammerdb>diset tpch pg_tpch_superuserpass xxxxxx
hammerdb>diset tpch pg_tpch_defaultdbase postgres
hammerdb>diset tpch pg_tpch_user postgres
hammerdb>diset tpch pg_tpch_pass xxxxxx
hammerdb>diset tpch pg_tpch_dbase tpch
# スケーリングファクターを100に設定 (約10GB)
hammerdb>diset tpch pg_scale_fact 10
# 並列実行数を指定(8並列)
hammerdb>diset tpch pg_num_tpch_threads 8
```
TPC-Cと同様にベンチマーク実行ユーザーを作成し、ベンチマークを実行する。
```hammerdbcli
hammerdb>vudestroy
hammerdb>loadscript
hammerdb>vuset vu 8
hammerdb>vuset logtotemp 1
hammerdb>vuset showoutput 1
hammerdb>vuset unique 1
hammerdb>vuset timestamps 1
hammerdb>print vuconf
Virtual Users = 8
User Delay(ms) = 500
Repeat Delay(ms) = 500
Iterations = 1
Show Output = 1
Log Output = 1
Unique Log Name = 1
No Log Buffer = 0
Log Timestamps = 1

hammerdb>vucreate
hammerdb>vurun
...
Vuser 1:Geometric mean of query times returning rows (22) is 8.35650
Vuser 2:Geometric mean of query times returning rows (22) is 9.22730
Vuser 3:Geometric mean of query times returning rows (22) is 10.44511
Vuser 4:Geometric mean of query times returning rows (22) is 9.00088
Vuser 5:Geometric mean of query times returning rows (22) is 10.95125
Vuser 6:Geometric mean of query times returning rows (22) is 8.22077
Vuser 7:Geometric mean of query times returning rows (22) is 9.40581
Vuser 8:Geometric mean of query times returning rows (22) is 10.69690
...
ALL VIRTUAL USERS COMPLETE
TPROC-H Driver Script
Benchmark Run jobid=66134D38615803E283634313
```

## CloudSQL for PostgreSQLの性能検証
### TPC-C Like
AlloyDBで実行したものと同様の手順で検証する。
```hammerdbcli
hammerdb>buildschema
```
ベンチマークを実行するユーザーと、ログを設定する。
```hammerdbcli
hammerdb>vudestroy
hammerdb>loadscript
hammerdb>vuset vu 8
hammerdb>vuset logtotemp 1
hammerdb>vuset showoutput 1
hammerdb>vuset unique 1
hammerdb>vuset timestamps 1
hammerdb>print vuconf
Virtual Users = 8
User Delay(ms) = 500
Repeat Delay(ms) = 500
Iterations = 1
Show Output = 1
Log Output = 1
Unique Log Name = 1
No Log Buffer = 0
Log Timestamps = 1
```
ベンチマーク実行ユーザーを作成する。
```hammerdbcli
hammerdb>vucreate
```
ベンチマークを実行する。
```hammerdbcli
hammerdb>vurun

```
### TPC-H Like

