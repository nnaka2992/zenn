---
title: "Cloud SQL for PostgreSQLのベクトル検索を試す"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud","db","postgresql", "LLM"]
published: true
---

Google Cloud Next '24でGoogle Cloudが提供するすべてのマネージドデータベースにベクトル検索の機能が追加されました。^[Cloud SQL for SQL Serverは除く]

今回はそのなかのCloud SQL for PostgreSQLにフォーカスしてベクトル検索機能を試します。

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
- 接続
  - パブリックIPを有効化

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
    CREATE ROLE app WITH LOGIN PASSWORD '**********';
    GRANT CREATE, CONNECT ON DATABASE vector_search TO app;
    GRANT ALL ON SCHEMA toys TO app;
    GRANT ALL ON SCHEMA public TO app;
    ALTER USER app SET search_path TO toys, public;
    
    ```

## `pgvector`を有効化する

以下のクエリを実行して、`pgvector`を有効化します。

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

`pgvector`を有効化するのに、データベースのフラグを編集する必要はありません。

## ベクトルを試す

実行環境はGoogle Cloudではない、[Google Colaboratory](https://colab.research.google.com/)を利用します。^[ローカルのPython環境で十分ですが、ローカルでPython環境を維持したくないのでColaboratoryを利用しています。]

:::message
*pgvectorにおけるインデックス*
Google Cloud for PostgreSQLで利用可能な`pgvector`のバージョンは0.6.0です。^[2024/05/26現在]
このバージョンの`pgvector`では以下のインデックスが利用可能です。

1. HNSWインデックス

   HNSWインデックスはインデックスの構築時間とメモリ使用量が多い一方で、問い合わせの応答が速くリコール^[偽陰性のようなもの]も少ない。

2. IVFFLatインデッス

   HNSWに比べてインデックス構築時間、メモリ使用量の両方で優れたパフォーマンスを示すものの、問い合わせとリコールの両方で劣る。

HNSWとIVFFlatはそれぞれリソース効率と性能でメリットとデメリットがあるため、ユースケースに応じて適切なインデックス手法を採用することが重要です。
:::

1. 使用するテーブルを作成する。

    今回はおもちゃの情報を保存するテーブルと、おもちゃの説明とそのベクトルデータを保存するテーブルの2つを作成する。
    ```sql
    CREATE TABLE products(
        product_id VARCHAR(1024) PRIMARY KEY,
        product_name TEXT,
        description TEXT,
        list_price NUMERIC
    );
    CREATE TABLE product_embeddings(
        product_id VARCHAR(1024) NOT NULL REFERENCES products(product_id),
        content TEXT,
        embedding vector(768)
    );
    ```
2. データを挿入する。

    おもちゃの説明データをPython経由でデータベースへ登録する。
    単純に[リンク](https://github.com/GoogleCloudPlatform/python-docs-samples/raw/main/cloud-sql/postgres/pgvector/data/retail_toy_dataset.csv)のデータを投入するのみのため省略する。
    詳細は[ノートブック](https://github.com/nnaka2992/vector_search_sample/blob/main/vector_search.ipynb)を参照。
    
   
3. ベクトル化データを生成し、データを挿入する。

    2で投入した一データのおもちゃの説明をVertexAIのEmbedding APIを利用してベクトル化する。
    ベクトル化する部分のコードサンプルは以下。詳細は[ノートブック](https://github.com/nnaka2992/vector_search_sample/blob/main/vector_search.ipynb)を参照。

    :::details サンプルコード
    ```python
    from langchain.text_splitter import RecursiveCharacterTextSplitter
    from langchain.embeddings import VertexAIEmbeddings
    from google.cloud import aiplatform
    import time
    import numpy as np
    from pgvector.asyncpg import register_vector
    
    # テキストをLLM APIのリクエストサイズ制限内に収まるように分割する。
    text_splitter = RecursiveCharacterTextSplitter(
        separators=[".", "\n"],
        chunk_size=500,
        chunk_overlap=0,
        length_function=len,
    )
    chunked = []
    for index, row in df.iterrows():
        product_id = row["product_id"]
        desc = row["description"]
        splits = text_splitter.create_documents([desc])
        for s in splits:
            r = {"product_id": product_id, "content": s.page_content}
            chunked.append(r)
    
    # 分割したテキストデータのベクトルデータを生成する。
    aiplatform.init(project=f"{PROJECT_ID}", location=f"{REGION}")
    embeddings_service = VertexAIEmbeddings()
    
    # VertexAIへのAPIアクセスに失敗した場合のリトライバックオフを定義する。
    def retry_with_backoff(func, *args, retry_delay=5, backoff_factor=2, **kwargs):
        max_attempts = 10
        retries = 0
        for i in range(max_attempts):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                print(f"error: {e}")
                retries += 1
                wait = retry_delay * (backoff_factor**retries)
                print(f"Retry after waiting for {wait} seconds...")
                time.sleep(wait)
    
    # ベクトルデータを生成する。
    batch_size = 5
    for i in range(0, len(chunked), batch_size):
        request = [x["content"] for x in chunked[i : i + batch_size]]
        response = retry_with_backoff(embeddings_service.embed_documents, request)
        # 生成したベクトルデータを保存する。
        for x, e in zip(chunked[i : i + batch_size], response):
            x["embedding"] = e
    
    # 取得したベクトルデータをデータフレームとして整形する。
    product_embeddings = pd.DataFrame(chunked)
    ```
    :::details

4. ベクトル検索のインデックスを作成する。

    前述のとおり`pgvector`では2種類のインデックスが利用できため、それぞれを利用したインデックスを作成する。
    ```sql
    CREATE INDEX ON product_embeddings USING hnsw(embedding vector_cosine_ops);
    CREATE INDEX ON product_embeddings USING ivfflat(embedding vector_cosine_ops);
    ```

    インデックスのパラメータにはそれぞれ以下の値を設定する必要がある。
    - HNSW
      - operator

        インデックスで使用する距離関数を指定する。
        詳細は後述。
      - m

        接続できるノードの最大数を表すパラメータ。
        mを大きくすると検索制度があがるが、検索時間とメモリ使用量が増える。
      - ef_construction

        インデックスの構築時に探索されるエントリポイント数。
        ef_constructionの値を大きくすると検索精度があがるが、検索時間とインデックス構築時間が増える。
    - IVFFlat
      - operator

        インデックスで使用する距離関数を指定する。
        詳細は後述。
      - lists

        IVFFlatが作成するクラスタの数を表すパラメータ。

    :::message
    `pgvector`で利用できる距離関数は以下のとおり。
    - L2距離
      - `vector_l2_ops`
    - 内積
      - `vector_ip_ops`
    - コサイン距離
      - `vector_cosine_ops`
    - L1距離
      - `vector_l1_ops`
      - `>=0.7.0`でのみ利用可
      - IVFFlatでは利用不可
    - ハミング距離
      - `bit_hamming_ops`
      - `>=0.7.0`でのみ利用可
      - IVFFlatでは利用不可
    - ジャガード距離
      - `bit_jaccard_ops`
      - `>=0.7.0`でのみ利用可
      - IVFFlatでは利用不可
    :::

# まとめ

今回はCloudSQL for PostgreSQLで利用可能な`pgvector`とVertexAIのEmbedded APIを使用てベクトル検索を試しました。
`pgvector`はPostgreSQLにおけるベクトル検索のデファクトと言えるほど普及しており、ほとんどのマネージドサービスで利用でき、またセルフホストした環境でも利用できます。

また利用方法も単純なのですでにCloud SQL for PostgreSQLを利用している組織でトランザクションシステム向けの用途に向いています。

# 参考
<!-- ignore-start -->
- [pgvectorの紹介](https://www.sraoss.co.jp/tech-blog/pgsql/pgvector-intro/)
- [5-11. レコメンドシステムではなぜユークリッド距離ではなくコサイン類似度が用いられるのか | Vignette & Clarity（ビネット＆クラリティ）](https://vigne-cla.com/5-11/)
- [近傍探索におけるユークリッド距離, cosine類似度, 内積は互いに変換可能](https://cympfh.cc/aiura/similarity-innnerproduct-l2-cosine)
- [RAGに捧げるベクトル検索パフォーマンスチューニング - 電通総研 テックブログ](https://tech.dentsusoken.com/entry/parameter_tuning_for_rag)
<!-- ignore-end -->
