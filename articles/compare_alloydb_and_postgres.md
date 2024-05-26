---
title: "CloudSQL for PostgreSQLのベンチマークと比較して理解するAlloyDBの特徴"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud","db","postgresql"]
published: true
---
# 概要
Google Cloudが提供するPostgreSQL互換データベースであるAlloyDBのパフォーマンスをトランザクション用途・分析用途の双方から検証する。
今回の検証ではAlloyDBの上限を見定めるのではなく、CloudSQLと比べてどのようなパフォーマンスになるを目的とする。

### TL;DR
- 絞り込み条件がインデックスに限定されない場合、AlloyDBのパフォーマンスメリットが特に大きくなる。
- 絞り込み条件がインデックスに限定され、かつデータサイズが小さい場合、CloudSQL for PostgreSQLのコストパフォーマンスが大きくなる。
- 現将・将来のワークロードの両方から利用するデータベースを検討すると良い。

# 検証方法
同一VPC上でプロビジョニングしたAlloyDBとCloudSQL for PostgreSQLに対して、Compute Engine上のHammerDBからTPC-C LikeなベンチマークとTPC-H Likeなベンチマークを実行する。
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
:::message <!-- ignore -->
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

## TPC-C Like
### 設定

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
# ロードしたデータ全てを対象にベンチマークを取得するよう設定
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
### AlloyDBの性能検証
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

### CloudSQL for PostgreSQLの性能検証
```hammerdbcli
hammerdb>vurun
...
Vuser 1:TEST RESULT : System achieved 39526 NOPM from 91194 PostgreSQL TPM
Vuser 1:FINISHED SUCCESS
Vuser 6:FINISHED SUCCESS
Vuser 7:FINISHED SUCCESS
Vuser 9:FINISHED SUCCESS
Vuser 4:FINISHED SUCCESS
Vuser 2:FINISHED SUCCESS
Vuser 3:FINISHED SUCCESS
Vuser 5:FINISHED SUCCESS
Vuser 8:FINISHED SUCCESS
ALL VIRTUAL USERS COMPLETE
Benchmark Run jobid=661354A3615803E213633313
```

### 結果
HammberDBのTPC-C likeはよくあるECサイトを模したワークロードになっている。
なので結果はさばいた新規注文数(NOPM)とPostgreSQLトランザクション数(TPM)という形で表現される。

- AlloyDB
  - 新規発注: 39,526件
  - トランザクション: 91,194件
- CloudSQL for PostgreSQL
  - 新規発注: 18,664件
  - トランザクション: 43,386件

この結果だけみるとCloudSQL for PostgreSQLがAlloyDBより優れた結果をだしていることが分かる。

これは主にCloudSQL for PostgreSQLで有効にしているデータキャッシュ(計算リソースインスタンスのローカル配置されたSSD)の影響受けているのではないかと考えられる。
今回のケースではデータセットが数十GB程度なのでデータがすべて計算リソースのインスタンス上に載るため、ネットワークディレイを大きく削減できたのではないかと推察される。

## TPC-H Like
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
```
ベンチマーク実行ユーザーを作成する。
```hammerdbcli
hammerdb>vucreate
```
### AlloyDBの性能検証
ベンチマークを実行する。
```hammerdbcli
hammerdb>vurun
...
ALL VIRTUAL USERS COMPLETE
TPROC-H Driver Script
Benchmark Run jobid=66134D38615803E283634313
```

### CloudSQL for PostgreSQLの性能検証
6時間程度実行してもCPUが100パーセントに貼りついたままクエリが完了せず。

### 結果
本来であれば各分析ワークロードのSQLの実行にかかった時間の平均を比較するべきだが、CloudSQL for PostgreSQLではクエリが完了しなかった^[AlloyDBでは長くても1分以内に完了]。

完了しなかったクエリは以下。
```sql
select 
  s_name, 
  s_address 
from 
  supplier, 
  nation 
where 
  s_suppkey in (
    select 
      ps_suppkey 
    from 
      partsupp 
    where 
      ps_partkey in (
        select 
          p_partkey 
        from 
          part 
        where 
          p_name like ':1%'
      ) 
      and ps_availqty > (
        select 
          0.5 * sum(l_quantity) 
        from 
          lineitem 
        where 
          l_partkey = ps_partkey 
          and l_suppkey = ps_suppkey 
          and l_shipdate >= date ':2' 
          and l_shipdate < date ':2' + interval '1 year'
      )
  ) 
  and s_nationkey = n_nationkey 
  and n_name = ':3' 
order by 
  s_name
```
対象となるテーブルはPKを除きインデックスが貼られていない。
そのためテーブルどうしを結合する際、不要なカラムもすべてメモリに読み込む必要があり、64GBしかないメモリを消費しつくしてしまったと推察できる。

一方でAlloyDBではカラムナエンジンを利用してインデックスが貼られていないカラムどうしの比較をうまく実行したため、本来であれば著しくパフォーマンスを低下させるクエリを実行できたと考えられる。

## まとめ
AlloyDBはCloudSQL for PostgreSQLに比べ、特にインデックスが貼られていないカラムを検索条件・結合条件などに指定した場合にパフォーマンスが高くなると分かった。^[カラムナストレージの特徴とも合致する]

一方で検索条件などの殆んどをインデックス経由で問い合わせしたデータサイズの小さいワークロードでは、Cloud SQL for Enterprise Plusでデータキャッシュを有効化するとコストパフォーマンスが高くなると言うことができる。^[今回のケースからデータサイズの大きいワークロードについては名言できない。]

AlloyDBかCloudSQL for PostgreSQLを選定する必要がある場合、上記を参考に現在・将来のワークロードから利用するデータベースサービスを考えるとよい。

## 参考
- [HammerDBをCLIで使うなど（８）：PostgreSQLにTPC-Hを実行してみる](https://atsuizo.hatenadiary.jp/entry/2019/09/04/090000)
- [TPC-Council/HammerDB: HammerDB Database Load Testing and Benchmarking Tool](https://github.com/TPC-Council/HammerDB)
- [データ キャッシュの概要  |  Cloud SQL for MySQL  |  Google Cloud](https://cloud.google.com/sql/docs/mysql/data-cache?hl=ja) 
