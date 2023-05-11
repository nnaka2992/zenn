---
title: "TiDBで学ぶNewSQLのアーキテクチャ for Beginners"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NewSQL", "DB", "TiDB"]
published: true
---

## はじめに
この記事ではNewSQLの特徴であるノード間の分散とトランザクションや分断耐性などがTiDBではどのような
技術によって実現されているかを説明することを目的としています。

Spannerの論文が2012年に発表されてから10年以上の年月が流れ、優れた論文や実装ドキュメント、個人による解説ブログなど
技術的詳細について述べた資料は多くあります。
加えてこの記事を入門的なものと位置づけているため各コンポーネントを網羅的に解説するというよりは、キーコンセプトを
どのように実装しているのかを実験を混じえながら動作の実現方法の解説を中心に扱います。

また今回はTiDBをベースに説明しますが、とくに分散と分散トランザクションについて検証します。

## NewSQLのコンポーネント
NewSQLのコンポーネントを網羅的に解説はしないと書いたものの、まったく書かないと説明も理解も難しくなるため
軽く紹介のみします。

### ストレージサーバ
TiDBではストレージサーバとして[TiKV]()というTiDBを開発しているPingCAPがRustで実装し現在は
Cloud Native Computing Foundationに寄贈されたKVSを採用しています。<br/>
現代のNewSQLのベースとなったSpannerと同様にストレージとDBノードと完全に分離しています。

### TiDBサーバ
TiDBではTiDBサーバがクエリリクエストの受信からクエリプランの生成、クエリ実行といった
SQLの処理を担当します。
他のNewSQLプロダクトに比べてTiDBがユニークな点としてクエリエンジンは、
完全にステートレスになっています。そのため容易にスケールアウトすることが出来ます。

### プレースメントドライバサーバ
プレースメントドライバ(PD)サーバはクラスタ全体のメタデータの管理を行ないます。
PDはクラスタのノード間の負荷分散やノードのステータス管理、クエリスケジューリングの最適化、
TiKVのノード情報やデータ容量・統計情報といったメタデータを管理します。

![TiDBのアーキテクチャ](https://download.pingcap.com/images/docs/tidb-architecture-v6.png)

## 実験と解説
### 実験環境
実験はローカルに建てた検証環境で行ないます。
検証環境は以下のように建てており、コマンドから分かる通りTiKV、TiDBサーバ、PDそれぞれ3っつずつ
起動しています。
```
$ tiup playground v6.5.0 --db 3 --pd 3 --kv 3
tiup is checking updates for component playground ...
Starting component `playground`: /home/nnaka2992/.tiup/components/playground/v1.12.1/tiup-playground v6.5.0 --db 3 --pd 3 --kv 3
Start pd instance:v6.5.0
Start pd instance:v6.5.0
Start pd instance:v6.5.0
Start tikv instance:v6.5.0
Start tikv instance:v6.5.0
Start tikv instance:v6.5.0
Start tidb instance:v6.5.0
Start tidb instance:v6.5.0
Start tidb instance:v6.5.0
Waiting for tidb instances ready
127.0.0.1:4000 ... Done
127.0.0.1:4001 ... Done
127.0.0.1:4002 ... Done
Start tiflash instance:v6.5.0
Waiting for tiflash instances ready
127.0.0.1:3930 ... Done

🎉 TiDB Cluster is started, enjoy!

Connect TiDB:   mysql --host 127.0.0.1 --port 4000 -u root
Connect TiDB:   mysql --host 127.0.0.1 --port 4002 -u root
Connect TiDB:   mysql --host 127.0.0.1 --port 4001 -u root
TiDB Dashboard: http://127.0.0.1:2379/dashboard
Grafana:        http://127.0.0.1:3000

```

### NewSQLと分散
NewSQLでは簡単にスケールイン/アウトが可能です。
さっそく実験してみましょう。
実験を簡素化するために以下のようなshellスクリプトを作成します。
動作は簡単で引数で指定したhostとportでアクセスしたノード上にあるテーブルに対して、
ランダムに生成した数字を挿入しています。
```
#!/usr/bin/env bash

host="$1"
port="$2"
uuid=$(uuidgen)
num=`expr 1 + $RANDOM % 1000`

mysql --host ${host} --port ${port} -u root --database test_db -e "INSERT INTO t_data(uuid, num) VALUES(UUID_TO_BIN('${uuid}'), ${num});"

if [ $? -ne 0 ]; then
    echo "insert failed"
    exit 1
fi
echo "insert uuid: ${uuid}, num: ${num} from ${host}:${port}"
```
実際に動かしてみると存在しないインスタンスであるport 4003を指定したものだけ失敗します。
```
$ ./insert.sh 127.0.0.1 4000
insert uuid: b6101c86-1537-401b-a3f8-a7c4728c4f90, num: 218 from 127.0.0.1:4000

$ ./insert.sh 127.0.0.1 4001
insert uuid: b665633a-f5bc-4af2-97e6-3132f5c741a8, num: 806 from 127.0.0.1:4001

$ ./insert.sh 127.0.0.1 4002
insert uuid: 8a931ace-e5d1-4d1a-a288-f1d8a303e13d, num: 667 from 127.0.0.1:4002

$ ./insert.sh 127.0.0.1 4003
ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1:4003' (111)
insert failed

```
ここから分かることは従来のPrimary/Secondary構成のRDBMSでは実現出来なかったマルチライター構成が実現出来ています。
またスケールアウトコマンドを実行すると簡単にライターインスタンスを増やすことが出来ます。
```
$ tiup playground scale-out --db 1
tiup is checking updates for component playground ...
Starting component `playground`: /home/nnaka2992/.tiup/components/playground/v1.12.1/tiup-playground scale-out --db 1
To connect new added TiDB: mysql --comments --host 127.0.0.1 --port 39599 -u root -p (no password)

$ ./insert.sh 127.0.0.1 39599
insert uuid: 41900a51-d6d0-4da5-a4c0-69a430b54a1b, num: 619 from 127.0.0.1:39599
```

この柔軟な分散の秘密はTiKVで実装されたストレージエンジンにあります。
TiKVではさらに下位のストレージエンジンとしてRocksDBを採用しています。
RocksDBはFacebookにより開発されたKVSデータベースです。

しかしRocksDB自体にはデータを分散する機能はなく、TiKVではRocksDB上にRaftプロトコルによりデータ分散を実現しています。
これによりTiKVは一つのインスタンスで障害が発生してもデータロストが発生しない、信頼性の高いスケールアウトするストレージを実現しています。

### NewSQLと分散トランザクション
TiDBでは従来のシャードベースの分散RDBと異なりDBノード間のトランザクションを自動で管理してくれるため、手動の分散トランザクション管理が不要です。
さっそく実験してみましょう。
実験は以下の通りです。

1. 分散トランザクションの観測に用いるレコードを作成する
1. トランザクションを開始する
1. 1で作成したレコードを`UPDATE`する
1. あたらしいセッションを開く(ログとして分かり易いようにsystemコマンドからmysqlqコマンドを実行)
1. 4で開いたセッションから1で作成したレコードを`SELECT`する
```
$ mysql --host 127.0.0.1 --port 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 142703
Server version: 5.7.25-TiDB-v6.5.0 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql> use test_db
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> INSERT INTO t_data(uuid, num) VALUES(UUID_TO_BIN('8c126efb-8503-4533-aa85-fbf2215e9964'), 111);
Query OK, 1 row affected (0.00 sec)

mysql> SELECT BIN_TO_UUID(uuid) as uuid, num from t_data where uuid = UUID_TO_BIN('8c126efb-8503-4533-aa85-fbf2215e9964');
 +--------------------------------------+-----+
| uuid                                 | num |
+--------------------------------------+-----+
| 8c126efb-8503-4533-aa85-fbf2215e9964 | 111 |
+--------------------------------------+-----+
1 row in set (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> UPDATE t_data SET num = 222 WHERE uuid = UUID_TO_BIN('8c126efb-8503-4533-aa85-fbf2215e9964');
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

> mysql> system mysql --host 127.0.0.1 --port 4001 -u root # Open New Connection!
> Welcome to the MySQL monitor.  Commands end with ; or \g.
> Your MySQL connection id is 415
> Server version: 5.7.25-TiDB-v6.5.0 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible
> 
> Copyright (c) 2000, 2023, Oracle and/or its affiliates.
> 
> Oracle is a registered trademark of Oracle Corporation and/or its
> affiliates. Other names may be trademarks of their respective
> owners.
> 
> Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
> 
> mysql> SELECT BIN_TO_UUID(uuid) as uuid, num from t_data where uuid = UUID_TO_BIN('8c126efb-8503-4533-aa85-fbf2215e9964');
> 
> mysql> use test_db;
> Reading table information for completion of table and column names
> You can turn off this feature to get a quicker startup with -A
> 
> Database changed
> mysql> SELECT BIN_TO_UUID(uuid) as uuid, num from t_data where uuid = UUID_TO_BIN('8c126efb-8503-4533-aa85-fbf2215e9964');
> +--------------------------------------+-----+
> | uuid                                 | num |
> +--------------------------------------+-----+
> | 8c126efb-8503-4533-aa85-fbf2215e9964 | 111 |
> +--------------------------------------+-----+
> 1 row in set (0.00 sec)
> 
> mysql> exit;
> Bye
mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)

> mysql> system mysql --host 127.0.0.1 --port 4001 -u root # Open New Connection!
> 
> Welcome to the MySQL monitor.  Commands end with ; or \g.
> Your MySQL connection id is 417
> Server version: 5.7.25-TiDB-v6.5.0 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible
> 
> Copyright (c) 2000, 2023, Oracle and/or its affiliates.
> 
> Oracle is a registered trademark of Oracle Corporation and/or its
> affiliates. Other names may be trademarks of their respective
> owners.
> 
> Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
> 
> mysql> use test_db;
> Reading table information for completion of table and column names
> You can turn off this feature to get a quicker startup with -A
> 
> Database changed
> mysql> SELECT BIN_TO_UUID(uuid) as uuid, num from t_data where uuid = UUID_TO_BIN('8c126efb-8503-4533-aa85-fbf2215e9964');
> +--------------------------------------+-----+
> | uuid                                 | num |
> +--------------------------------------+-----+
> | 8c126efb-8503-4533-aa85-fbf2215e9964 | 222 |
> +--------------------------------------+-----+
> 1 row in set (0.00 sec)
```
このログから分かるようにセッションを跨いで、トランザクションが管理されていることが分かります。

この分散トランザクションはTiKVのRaftプロトコルとPDによるトランザクション管理、
そして2層コッミットにより支えられています。

## まとめ
NewSQLではこれまでのデータベースソリューションでは困難だったり複雑な処理の設計が必要だった、データの分散と分散トランザクションという課題を解決しています。
時間とスペースの関係で触れませんでしたが、TiDBにはTiSparkやTiFlashなど分析ワークロードに適した機能が複数あります。
これはSpannerや他のNewSQLにはない優れた機能です。

## 参考
- [TiDBアーキテクチャ](https://docs.pingcap.com/ja/tidbcloud/tidb-architecture)
- [TiDB ストレージ](https://docs.pingcap.com/ja/tidbcloud/tidb-storage)
- [TiDB コンピューティング](https://docs.pingcap.com/ja/tidbcloud/tidb-computing)
- [TiDB: A Raft-based HTAP Database](http://www.vldb.org/pvldb/vol13/p3072-huang.pdf)
