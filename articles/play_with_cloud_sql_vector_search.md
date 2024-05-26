---
title: "Cloud SQLのベクトル検索を試す"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

Google Cloud Next '24でGoogle Cloudが提供するすべてのマネージドデータベースにベクトル検索の機能が追加されました。^[Cloud SQL for SQL Serverは除く]

今回はそのなかのCloud SQLにフォーカスしてベクトル検索機能を試します。

# Cloud SQL for PostgreSQL

## インスタンススペック
- エディション
  - Enterprise
- vCPU
  - 2
- RAM
  - 8GB
- ストレージタイプ
  - SSD
- Zone
  - asia-northeast1

## 必要な設定を行う

1. データベースを作成する

    以下のクエリを実行し、データベースを作成する。

    ```sql
    CREATE DATABASE vector_search;
    ```

2. スキーマを作成する

    作成したデータベースに接続し、以下のクエリを実行してtoysスキーマを作成する。

    ```sql
    CREATE SCHEMA toys;
    ```

3. データベースユーザーを作成する

    アプリケーションからデータベース接続するユーザーを作成し、必要な権限を付与する。

    ```sql
    CREATE ROLE app WITH PASSWORD '**********';
    GRANT CONNECT ON DATABASE vector_search TO app;
    ALTER DEFAULT PRIVILEGES IN SCHEMA toys GRANT SELECT, INSERT, UPDATE, DELETE, REFERENCES ON TABLES TO app;
    ```

## `pgvector`を有効化する

以下のクエリを実行して、`pgvector`を有効化します。

```sql
CREATE EXTENSION IF NOT EXISTS vector;;
```

`pgvector`を有効化するのに、データベースのフラグを編集する必要はありません。

## ベクトルを試す




# Cloud SQL for MySQL



今後。

MySQLはPostgreSQLと異なり、ベクトル検索に対応するメジャーなプラグインはありません。 ^[2024/05/24現在]

そのためメジャークラウドで提供されているすべてのマネージドMySQLがベクトル検索機能をサポートしているわけではありません。


