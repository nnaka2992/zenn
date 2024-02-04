---
title: "YugabyteDB ManagedのAlways Free枠を試そう"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['DB']
published: true
---
YugabyteDB Managedにフリートライアルがあるのは知っていたのですが、期間が限られたものしか無いと思っていました。
YugabyteDBについて調べごとをしていたら機能制限はあるもののSandboxクラスターというクレジットカード登録すら不要でAlways Freeな利用枠があることを知りました。

いままでローカルでYugabyteDBを建てたりminikube上で遊んでいたのですが、簡単な検証であればSandboxクラスターで十分です。
この記事ではそんなYugabyteDB ManagedのSandboxクラスターを紹介します。

# Sandbox Clusterの制限
Snadboxクラスターはその名の通りお試し用のクラスターです。
そのためDedicatedクラスターという商用利用を前提とした課金が必要なクラスターに比べて機能が制限されます。

具体的に制限される機能は以下です。
- 1アカウントにつき1クラスターのみ
- シングルノードのみ
- 2vCPU/4GB Memory/10GB Storage/500テーブルまたは12.5M行/15コネクション
- 自動バックアップ無効
- 占有VPC利用不可
- SLA無し

商用利用することを考えるとシングルノードではYugabyteDBを選定するメリットが少いですし、ノード・テーブル関連の制限も厳しいものがあります。
しかし操作感を確認するには十分でしょう。

# Sandbox Clusterの作成

## アカウントを作成する
[YugabyteDB Managedのサインアップページ](https://cloud.yugabyte.com/signup)からアカウントを作成できます。

アカウントの作成にはGoogle・Github・LinkedInのアカウント、またはメールアドレスよるアカウントの作成が選択できます。

サインアップ完了後はSandboxクラスターを作成出来るページに遷移します。

![toppage](/images/play_with_yugabytedb_managed_sandbox/2_1_1_toppage.png)

## クラウターを作成する
上記画像にある`Create a Free cluster`を押下するとクラスタータイプを選択できる画面に遷移するのでSandboxクラスターを選択します。
設定画面は`CLUSTER SETTINGS`と`NETWORK ACCESS`、 `DB CREDENTIALS`の3画面に分れています。

### `CLUSTER SETTINGS`
一番最初の`CLUSTER SETTINGS`の画面では以下の4項目の設定を行います。
1. クラスタ名
2. クラウドプロバイダー
   AWSかGCPかを選択できる。DedicatedクラスタであればAzureも選択出来ます。
3. リージョン
   AWSとGCPそれぞれのリージョンを選択出来ます。少くとも東京と大阪は選択出来ます。
4. データベースバージョン
   SandboxクラスターではPreview Releaseのみが対象です。

![cluster settings](/images/play_with_yugabytedb_managed_sandbox/2_2_1_cluster_settings.png)
### `NETWORK ACCESS`
`NETWORK ACCESS`ではIPのallow listを設定します。
パブリッククラウドでは様々な用途でネットワーク接続が行なわれるため、IPアドレスやCIDRの設定の他に許可するポートの指定などもありましたがYugabyteDBはデータベースとそれに関するコンソールのみを提供するサービスのため設定対象はIP/CIDRのみです。

![network access](/images/play_with_yugabytedb_managed_sandbox/2_2_2_network_access.png)
### `DB CREDENTIALS`
`DB CREDENTIALS`ではデータベースのパスワードを設定します。
ここではYugabyteDB Managedが自動で生成したパスワードを利用するか、YSQL/YCQLそれぞれにユーザーがパスワードを設定するかを選ぶことができます。
![db credentials](/images/play_with_yugabytedb_managed_sandbox/2_2_3_db_credentials.png)

ここまで進めるとデータベースの作成が開始され、5-10分ほど待つとYugabyteDBクラスターが利用可能になります[^1]。
![database ready](/images/play_with_yugabytedb_managed_sandbox/2_2_4_database_ready.png)

# YugabyteDBクラスターを使ってみる
## YugabyteDBクラスターに接続する
YugabyteDB Managedでクラスターに接続する方法は以下の3通りがあります。[^2]
1. [Cloud Shellから接続する方法](https://docs.yugabyte.com/preview/yugabyte-cloud/cloud-connect/connect-cloud-shell/)
2. [Client Shellから接続する方法](https://docs.yugabyte.com/preview/yugabyte-cloud/cloud-connect/connect-client-shell/)
3. [アプリケーションから接続する方法](https://docs.yugabyte.com/preview/yugabyte-cloud/cloud-connect/connect-applications/)

今回は最も簡単なCloud Shellから接続する方法を試します。

クラスターの詳細画面にある`Connect`を押下すると接続方法のリストが表示されるため、`Launch Cloud Shell`を選択します。
そうすると`DATABASE NAME`、`DATABASE USER NAME`、`API TYPE`を入力するように求められるためそれぞれ以下の用に入力します。
- `DATABASE NAME`: yugabyte
- `DATABASE USER NAME`: admin (またはDB CREDENTIALSの設定時に作成したデフォルトユーザー)
- `API TYPE`: YSQL

![launch_cloud_shell](/images/play_with_yugabytedb_managed_sandbox/3_1_1_launch_cloud_shell.png)
すると以下のような画面でデータベースパスワードの入力を求められるため、`DB CREDENTIALS`で設定または指定されたパスワードを入力します。
![enter password](/images/play_with_yugabytedb_managed_sandbox/3_1_2_enter_password.png)
ここまで完了すればYugabyteDBに対してSQLを実行出来るようになりました。

## YugabyteDBでSQLを実行する
YugabyteDB Managedで`Cloud Shell`を利用するとExplore YugabyteDB in 5 minutsなるコンテンツが用意されているのでそのSQLを試します。

### テーブルを作成する
以下の2つのDDLを実行してテーブルを作成します。
```sql
CREATE TABLE IF NOT EXISTS public.dept (
  deptno integer NOT NULL,
  dname text,
  loc text,
  description text,
  CONSTRAINT pk_dept PRIMARY KEY (deptno asc)
);

CREATE TABLE IF NOT EXISTS emp (
  empno integer generated by default as identity (start with 10000) NOT NULL,
  ename text NOT NULL,
  job text,
  mgr integer,
  hiredate date,
  sal integer,
  comm integer,
  deptno integer NOT NULL,
  email text,
  other_info jsonb,
  CONSTRAINT pk_emp PRIMARY KEY (empno hash),
  CONSTRAINT emp_email_uk UNIQUE (email),
  CONSTRAINT fk_deptno FOREIGN KEY (deptno) REFERENCES dept(deptno),
  CONSTRAINT fk_mgr FOREIGN KEY (mgr) REFERENCES emp(empno), 
  CONSTRAINT emp_email_check CHECK ((email ~ '^[a-zA-Z0-9.!#$%&''*+/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$'::text))
);
```
![run ddl](/images/play_with_yugabytedb_managed_sandbox/3_2_1_run_ddl.png)

PostgreSQLのようにPKの指定はもちろん、ユニークキーやFK、チェック制約が指定できることがわかります。

### INSERT文を実行する
続けて以下のDMLを実行し、テーブルに行を追加します。
```sql
INSERT INTO dept (deptno,  dname,        loc, description)
     VALUES    (10,     'ACCOUNTING', 'NEW YORK','preparation of financial statements, maintenance of general ledger, payment of bills, preparation of customer bills, payroll, and more.'),
            (20,     'RESEARCH',   'DALLAS','responsible for preparing the substance of a research report or security recommendation.'),
            (30,     'SALES',      'CHICAGO','division of a business that is responsible for selling products or services'),
            (40,     'OPERATIONS', 'BOSTON','administration of business practices to create the highest level of efficiency possible within an organization');

INSERT INTO emp (empno, ename,    job,        mgr,   hiredate,     sal, comm, deptno, email, other_info)
     VALUES   (7369, 'SMITH',  'CLERK',     7902, '1980-12-17',  800, NULL,   20,'SMITH@acme.com', '{"skills":["accounting"]}'),
            (7499, 'ALLEN',  'SALESMAN',  7698, '1981-02-20', 1600,  300,   30,'ALLEN@acme.com', null),
            (7521, 'WARD',   'SALESMAN',  7698, '1981-02-22', 1250,  500,   30,'WARD@compuserve.com', null),
            (7566, 'JONES',  'MANAGER',   7839, '1981-04-02', 2975, NULL,   20,'JONES@gmail.com', null),
            (7654, 'MARTIN', 'SALESMAN',  7698, '1981-09-28', 1250, 1400,   30,'MARTIN@acme.com', null),
            (7698, 'BLAKE',  'MANAGER',   7839, '1981-05-01', 2850, NULL,   30,'BLAKE@hotmail.com', null),
            (7782, 'CLARK',  'MANAGER',   7839, '1981-06-09', 2450, NULL,   10,'CLARK@acme.com', '{"skills":["C","C++","SQL"]}'),
            (7788, 'SCOTT',  'ANALYST',   7566, '1982-12-09', 3000, NULL,   20,'SCOTT@acme.com', '{"cat":"tiger"}'),
            (7839, 'KING',   'PRESIDENT', NULL, '1981-11-17', 5000, NULL,   10,'KING@aol.com', null),
            (7844, 'TURNER', 'SALESMAN',  7698, '1981-09-08', 1500,    0,   30,'TURNER@acme.com', null),
            (7876, 'ADAMS',  'CLERK',     7788, '1983-01-12', 1100, NULL,   20,'ADAMS@acme.org', null),
            (7900, 'JAMES',  'CLERK',     7698, '1981-12-03',  950, NULL,   30,'JAMES@acme.org', null),
            (7902, 'FORD',   'ANALYST',   7566, '1981-12-03', 3000, NULL,   20,'FORD@acme.com', '{"skills":["SQL","CQL"]}'),
            (7934, 'MILLER', 'CLERK',     7782, '1982-01-23', 1300, NULL,   10,'MILLER@acme.com', null);
    
```
![run dml](/images/play_with_yugabytedb_managed_sandbox/3_2_2_run_dml.png)

### SELECT文を実行する
最後に作成したテーブルに対してSELECT文を実行します。
```sql
SELECT
    dept.deptno,
    dept.dname,
    dept.loc,
    emp.empno,
    emp.ename,
    emp.job
FROM public.dept AS dept
INNER JOIN public.emp AS emp
    ON dept.deptno = emp.deptno;
```
普通にSELECT文も発行できますね。
![run select](/images/play_with_yugabytedb_managed_sandbox/3_2_3_run_seelct.png)

# まとめ
今回はYugabyteDB ManagedのSandboxクラスターを紹介しました。
前述の通りプロダクション用途での利用には向いていませんが、使用感を試すには十分ではないでしょうか？
惜しむらくは複数ノードでWriteスケールの機能を見られたらとうれしいです。

[^1]: と画面には書いてありましたが実際には15分くらいでした。気にするほどではないですが書かれている時間より長いとモヤモヤします。
[^2]: Managedではない場合2、3のみ
