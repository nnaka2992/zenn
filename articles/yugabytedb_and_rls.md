---
title: "大規模マルチテナントシステムのためのYugabyteDBによるRLSの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["db", "YugabyteDB", "PostgreSQL", "rls"]
published: false
---
分散SQLやNewSQLとよばれるデータベース製品はGoogleによるCloud Spannerや、PingCAPが中心となって開発するTiDB。CockroachLabが主体となって開発するCockroachDBなどどさまざまな製品が存在します。

TiDBはMySQL互換のNewSQL市場を独占していたり、CockroachDBはPostgreSQL互換の中では最高レベルのパフォーマンスを誇っていたり、SpannerはGoogleのサービスを支えてきたという強力な実績を持っていたりとそれぞれの製品にそれぞれの魅力があります。

これらのNewSQLと並んで有名な製品としてYugabyteDBというデータベースがありますが、ほかのデータベースに比べると国内での導入実績が存在せず、国内では影が薄いように感じます。^[少なくとも会社名を出した導入実績はないです。]

しかしそれはけっしてYugabyteDBがほかNewSQLに比べて特筆すべき点がない、劣ったデータベースということではありません。

この記事ではNewSQLの中でもYugabyteDBでしか実現できない機能として、マルチテナントシステムで利用するデータベースにはほぼ必須と言えるRow Level Security(RLS)の機能をPostgreSQL互換NewSQLで唯一サポートしているという点があります。^[「Amazon Aurora Limitless Databaseはどうなんだ。」という声が聞こえてきそうですが、Limitelessはプレビューなので対象外とします。]

この記事ではYugabyteDBにおけるRow Level Securityを紹介します。

# Row Level Securityのサポート

ここまで述べてきたPostgreSQLの互換性と分散SQL特有の特性は、大規模システムを支えるうえでYugabyteDBが優れていると言える特徴の一部です。
しかしこれらだけではCloud SpannerやCockroachDB、TiDBといったほかの分散SQLに比べ、YugabyteDBがマルチテナントシステムに向いていると結論付けることはできません。

分散SQLの特徴はもちろん、ほかの分散SQLも供えていますし、そもそもPostgreSQL互換であることは優れたデータベースの必須要件ではありません。

これらの分散SQLとYugabyteDBが一線を隔ている特徴はRow Level Security^[行レベルのセキュリティ]にあります。^[Cloud Spannerにはきめ細かいアクセス制御と呼ばれる機能がありますが、Row Level Securityは提供していません。]

## Row Level Securityとは

Row Level Securityとは一般的にRLSと略される機能で、問い合わせで変更、または取得できる行をデータベースユーザー単位で制御できます。

この機能を使うことでマルチテナントシステムにおけるデータの取り扱いをシンプルかつ安全に実行できます。

Row Level SecurityはANSI SQLの標準に含まれない機能ではあるものの、PostgreSQL以外にもOracle Dataabseや、BigQueryなど多様なDBMSでサポートされています。

Row Level Securityがサポートしていないデータベースの場合データベースやそれをホストするインスタンスレベルで分離するか、すべてのSQLにWHERE句指定してデータアクセスをコントロールする必要があります。
前者2つの方法であればまだしも、WHERE句を指定する方法はアプリケーションの実装を間違えると情報漏洩につながるため避けるほうが良いでしょう。

:::message
**マルチテナントの実装パターン**

マルチテナントシステムのデータベースを構築するばあい、その実装方法は以下の3通りに別れます。

1. 1データベースマルチテナント

   RDBMSでマルチテナントシステムを組むうえで最適解といえる構成です。

   ひとつのデータベースで全てのテナントのデータを扱うため、金銭コストや変更のリリースの複雑さを回避することができます。

   その一方である企業のデータをほか企業からアクセスできて^[いわゆるData Violation]しまったり、スケーラビリティに課題を抱えることがあります。

   しかし前者についてはRow Level Securityで、後者については分散SQLで解決可能です。
3. 1データベース1テナント

   ひとつのテナントに対してひとつのデータベース^[PostgreSQL(psql)の`\l`で表示できるものや、MySQLの`SHOW DATABASES`で表示できるもの]を割りあてます。

   そのため1の手法と同様に金銭コストは抑えることができますが、機能追加などによる変更を全てのデータベースに対して実行する必要があるためテナント数と比例してリリースコストがあがります。
4. 1インスタンス1テナント

   ひとつのテナントにひとつのインスタンスを割り当てる方法です。

   テナント間でリソースの共有ができないため、金銭コストが上昇するうえ、2の手法と同様に変更のリリースがテナント数と比例して複雑になります。

   一方でスケールアウトしないデータベースでも性能課題を抱えづらく、またインスタンスレベルでの障害が発生した場合の影響も最小限に抑えることができます。
:::

前述のPostgreSQLとの互換性に記したとおりYugabyteDBは、さまざまなPostgreSQLの機能を高レベルでサポートしています。
ここで記載したRow Level Securityもその例にもれず、YugabyreDBで利用できる機能となっています。

### Row Level Securityを利用する

実際にRow Level Securityを利用する方法を紹介します。
ここでは簡単のため、ID列にBIGSERIAL型を利用していますが、ホットスポットの原因となるため本番用途では推奨しません。

1. Row Level Securityを利用するテーブルを作成する

   まず始めにRow Level Securityを適用する対象のテーブルを作成します。
   ```sql
   CREATE TABLE companys (
       company_id BIGSERIAL NOT NULL,
       company_name varchar(255) UNIQUE NOT NULL,
       address text NOT NULL,
       PRIMARY KEY (company_id)
   );
   CREATE TABLE accounts (
       user_id BIGSERIAL UNIQUE NOT NULL,
       company_id BIGINT UNIQUE NOT NULL,
       company_name varchar(255) UNIQUE NOT NULL,
       name varchar(255) NOT NULL,
       email_address varchar(255) UNIQUE NOT NULL,
       PRIMARY KEY (user_id),
       FOREIGN KEY (company_id) REFERENCES companys (company_id)
   );
   ```
2. テーブルでRow Level Securityを利用可能にする

   次に作成したテーブルに対してRow Level Securityを利用できるようにします。
   ```sql
   ALTER TABLE companys ENABLE ROW LEVEL SECURITY;
   ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
   ```

3. Row Level Security用のポリシーを作成する
   
   続けてRow Level Securityとアカウントを紐付けるロールとポリシーを作成します。
   ```sql
   - three_shakeロールを作成し必要な権限を付与する
   CREATE ROLE three_shake WITH LOGIN PASSWORD 'TEST123';
   GRANT ALL ON TABLE companys, accounts TO three_shake;
   GRANT three_shake TO current_user;
   
   - three_shakeロールに対してPOLCYを設定する
   CREATE POLICY companys_three_shake ON companys TO three_shake USING (company_name = current_user);
   CREATE POLICY accounts_three_shake ON accounts TO three_shake USING (company_name = current_user);

   - sreakeロールを作成し必要な権限を付与する
   CREATE ROLE sreake WITH LOGIN PASSWORD 'TEST123';
   GRANT ALL ON TABLE companys, accounts TO sreake;
   GRANT sreake TO current_user;

   - sreakeロールに対してPOLCYを設定する
   CREATE POLICY companys_sreake ON companys TO sreake USING (company_name = current_user);
   CREATE POLICY accounts_sreake ON accounts TO sreake USING (company_name = current_user);
   ```

4. Row Level Securityが動作しているか確認する

   最後にRow Level Securityの動作を確認します。
   
   まず初めに検証用データを投入します。
   ```sql
   INSERT INTO companys (company_name, address) values ('three_shake','address_three_shake');
   INSERT INTO accounts (company_id, company_name, name, email_address) values (1, 'three_shake','user1', 'user1@three_shake.example.com');

   INSERT INTO companys (company_name, address) values ('sreake','address_sreake');
   INSERT INTO accounts (company_id, company_name, name, email_address) values (2, 'sreake','user2', 'user2@sreake.example.com');
   ```

   実際に登録されたデータは以下の通りです。

   **companys**

   | company_id | company_name | address             |
   |------------|--------------|---------------------|
   | 1          | three_shake  | address_three_shake |
   | 2          | sreake       | address_sreake      |

   **accounts**

   | user_id | company_id | company_name | name  | email_address                 |
   |---------|------------|--------------|-------|-------------------------------|
   | 1       | 1          | three_shake  | user1 | user1@three_shake.example.com |
   | 2       | 2          | sreake       | user2 | user2@sreake.example.com      |

   続いて投入したデータを取得します。

    ```sql
    SET ROLE three_shake;
    SELECT * FROM companys;
    SELECT * FROM accounts;
    ```

    出力されたデータは以下の通りです。
    
   **companys**

   | company_id | company_name | address             |
   |------------|--------------|---------------------|
   | 1          | three_shake  | address_three_shake |

   **accounts**

   | user_id | company_id | company_name | name  | email_address                 |
   |---------|------------|--------------|-------|-------------------------------|
   | 1       | 1          | three_shake  | user1 | user1@three_shake.example.com |

    
    SQLと出力結果から分かるようにWHERE句で絞りこみの条件を記載せずとも、自動で選択すべきでないデータは排除されています。
   

このようにYugabyteDBでは簡単な手順に沿ってテーブルと関連リソースを作成することで、Row Level Securityを利用できます。

またここで紹介した一連のSQLはPostgreSQLでもそのまま動作します。

# 引用

- “5.8. 行セキュリティポリシー.” n.d. Accessed May 8, 2024. https://www.postgresql.jp/document/15/html/ddl-rowsecurity.html.
- “Sql,Security: Support Row Level Security · Issue #73596 · Cockroachdb/Cockroach.” n.d. GitHub. Accessed May 8, 2024. https://github.com/cockroachdb/cockroach/issues/73596.
- Uqichi. 2019. “PostgreSQLのRow Level Securityを使ってマルチテナントデータを安全に扱う.” HRBrain Blog. June 24, 2019. Accessed May 8, 2024. https://times.hrbrain.co.jp/entry/postgresql-row-level-security.
- “YugabyteDB FAQS.” 2024. YugabyteDB Docs. January 29, 2024. https://docs.yugabyte.com/preview/faq/general#what-is-yugabytedb.
- “きめ細かいアクセス制御の概要.” n.d. Google Cloud. Accessed May 8, 2024. https://cloud.google.com/spanner/docs/fgac-about?hl=ja#postgresql.
