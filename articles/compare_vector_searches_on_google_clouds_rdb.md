---
title: "Google Cloud主催パートナー向けイベントで「Google Cloud で利用できるRDBのベクトル検索を徹底解剖！」を話しました。"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "DB", "PostgreSQL", "MySQL", "LLM"]
published: true 
published_at: 2024-06-10 08:00
---

2024年6月5日にGoogle Cloudがパートナー向けに開催したデータ関連の非公開イベントで<!-- ignore-start -->「Google Cloud で利用できるRDBのベクトル検索を徹底解剖！」<!-- ignore-end -->というLTを話しました。

https://speakerdeck.com/nnaka2992/google-cloud-deli-yong-dekirurdbnobekutorujian-suo-woche-di-jie-pou

非公開イベントのため録画がなかったり、LT枠だった関係で省略してしまった部分があったりしたためブログでより詳細な説明資料のようなものを書きました。

# 背景

Google Cloudが提供する全7種類のマネージドデータベースはサポート状況に違いがあるものの、すべてがベクトル検索をサポートしています。<!-- ignore-start -->^[Memorystore for memcachedやCloud SQL for SQL Serverなどエンジンレベルでは未対応のものもあります。]<!-- ignore-end -->

その中からリレーショナルデータベースに限定しても4種類のデータベースが存在し、細かなエンジンレベルでは5種類もの選択肢がありします。^[Cloud SQL、AlloyDB、Spanner、BigQueryの4種類。]^[インタフェースまで細分化するとSpannerがGoogle SQLとPostgreSQL互換の2インタフェースをもつため、6種類になります。]

この記事ではそれぞれのRDBとベクトル検索の特性を整理するとで、どのデータベースがどんなユースケースに向いているかを整理します。

# ベクトル検索とは

従来の検索手法が住所やID、商品名などのフィールドに完全一致や部分一致、または不一致などの条件で検索していました。
::: message
**従来の検索の例**<!-- ignore-start -->
- `id = '121'`
- `postal_code = '1008929'`
- `created_at BETWEEN SYSDATE AND SYSDATE - interval '1 day'`
- `description LIKE '%vector search%'`
<!-- ignore-end -->
:::

このような従来の検索手法では必要としている状態が分かりきっている場合や、明確に指定できる場合にはパフォーマンス的にも優れている場合が多いです。

しかし実世界では条件を完全に指定することが難しいことも多く、`よろこばれる誕生日プレゼント`や`おいしいお菓子`のように検索したいものごとの方向性しか指定できないケースが多々あります。

そういった検索条件に対応したのがベクトル検索です。
ベクトル検索とは近傍法^[Nearest Neighbor Algorithm]という古典的機械学習アルゴリズムを利用した検索手法です。
これは検索対象のベクトル郡と入力されたベクトルの距離を計算し、入力に最も近い結果を返します。これを平たく表現すると、検索条件に似た結果を返す検索方法と言えます。

::: message
**実際に使われているベクトル検索の例**<!-- ignore-start -->
- Google検索
- Google画像検索
- Mercariの商品推薦アルゴリズム
- ヤフオク!の出品商品画像の推定アルゴリズム
- Spotifyのポッドキャストの検索アルゴリズム^[自然言語による音声データの検索]
<!-- ignore-end -->
:::

これまでベクトル検索を利用したあいまい検索や意味検索は、高度な技術力を有する一部の企業でしか利用することが難しかったです。
しかし昨今の生成AIの発達やベクトル検索を利用可能なデータベースの出現で、あらゆる企業が自社のシステムやサービスに高度な検索手法を導入できるようになりました。

## アプリケーションがベクトル検索を実行される手順

実際にベクトル検索を利用するアプリケーションは以下のような手順で実行されます。

1. ベクトル検索対象のデータをベクトル化しデータベースに保存する。

   ![ベクトル検索対象のデータをベクトル化しデータベースに保存する。](/images/compare_vector_searches_on_google_clouds_rdb/how_app_works_1.png)
2. ユーザーからの問い合わせを受け取る。

    ![ユーザーからの問合せを受けとる。](/images/compare_vector_searches_on_google_clouds_rdb/how_app_works_2.png)

3. ユーザーからの問い合わせをベクトル化する。

    ![ユーザーからの問合せをベクトル化する。](/images/compare_vector_searches_on_google_clouds_rdb/how_app_works_3.png)

4. ベクトル化した問い合わせでデータベースを検索する。

    ![ベクトル化した問合せでデータベースを検索する。](/images/compare_vector_searches_on_google_clouds_rdb/how_app_works_4.png)

5. 問い合わせした内容に似た結果をユーザーに返す。

    ![問い合わせした内容に似た結果をユーザーに返す。](/images/compare_vector_searches_on_google_clouds_rdb/how_app_works_5.png)

# それぞれのデータベースにおけるベクトル検索の特色とユースケース

ベクトル検索やデータベースを選択するばあいに限らず、利用する製品を決定する場合はそれぞれにどのような特徴があるのかを把握することが大切です。
ここでもそれに習い、それぞれのデータベースの特徴を整理したうえでユースケースを考えていきます。

## Cloud SQL for PostgreSQL

### 特徴

- OSS版PostgreSQLとランタイムレベルで互換性がある。
  - ほかクラウドやオンプレミスでも同等の環境を再現しやすい。
- RLS^[行レベルセキュリティ]によるマルチテナントや細やかなアクセスコントロールが必要な場合の利用に向いている。
- ミリ秒単位のレイテンシ
- ベクトル検索をOSSのpgvectorで実装している。
  - PostgreSQL互換の製品であれば利用できることが期待できる。
  - 厳密な近傍検検索^[KNN]と近似最近傍探索^[ANN]をインデックスとして利用できる。
  - ベクトル間の距離を算出するアルゴリズムが多数存在する。
  - 開発が活発なOSSのため頻繁に機能が拡充している。
- SQLからVertexAIへのText Embedding API^[データをベクトル化するAPIのこと]を実行できる。

### ユースケース

- 将来Google Cloud以外環境への移行を検討する可能性があるばあい。
- Google Cloudをプライマリまたはレプリカとしたマルチプラットフォーム環境でベクトル検索を利用するばあい。
- ミリ秒単位のレイテンシが求められるばあい。
- Cloud SQL for PostgreSQL上に存在するデータに対してリアルタイム性の高いベクトル検索を実装したいばあい。
  - 商品情報テーブルの商品説明に対してあいまい検索する。
  - イベント情報テーブルのイベント説明に対してあいまい検索する。
- 新規でRAGアプリケーションを実装するばあい。
  - pgvector由来の高度なベクトル検索機能や、RLSによるベクトル化データへユーザーごとのアクセス権限付与が利用できるため。

## Cloud SQL for MySQL

### 特徴

- OSS版PostgreSQLとランタイムレベルで互換性がある。
  - ほかクラウドやオンプレミスでも同等の環境を再現しやすい。
- ミリ秒単位のレイテンシ
- Google Cloudで独自のベクトル検索機能を実装している。
  - OSSや特定のクラウドではベクトル検索が利用できない。
  - 複数クラウドのマネージドサービスで実装されているものの、構文がそれぞれ異なる。
    - 将来MySQL自体やプラグインでデファクトな実装がでてきたとき、負債になる可能性がある。
  - 厳密な近傍検検索をインデックスとして利用できる。
  - ベクトルインデックスはメモリを常時使用する。
  - 現在はPreviewステータス。

### ユースケース

- Cloud SQL for MySQL上に存在するデータに対してリアルタイム性の高いベクトル検索を実装したいばあい。
  - 商品情報テーブルの商品説明に対してあいまい検索する。
  - イベント情報テーブルのイベント説明に対してあいまい検索する。
  - 個人的に現時点ではネガティブな要素が多く、採用を控えるほうが懸命。

## AlloyDB for PostgreSQL

### 特徴

- 列指向エンジンによる集計やインデックスを張っていない列の検索でもパフォーマンスの劣化が少ない。
- RLSによるマルチテナントや細やかなアクセスコントロールが必要な場合の利用に向いている。
- ミリ秒単位のレイテンシ
- ベクトル検索をOSSのpgvectorで実装している。
  - PostgreSQL互換の製品であれば利用できることが期待できる。
  - 厳密な近傍検検索と近似最近傍探索をインデックスとして利用できる。
    - pgvectorがサポートしている上記インデックスに加え、SCaNNインデックスという独自のインデックスをサポートしている。
  - ベクトル間の距離を算出するアルゴリズムが多数存在する。
  - 開発が活発なOSSのため頻繁に機能が拡充している。
- SQLからVertexAIへのText Embedding APIを実行できる。

### ユースケース

- AlloyDB for PostgreSQL上に存在するデータに対してリアルタイム性の高いベクトル検索を実装したいばあい。
  - 商品情報テーブルの商品説明に対してあいまい検索する。
  - イベント情報テーブルのイベント説明に対してあいまい検索する。
- Cloud SQL for PostgreSQLではパフォーマンスが足りないと判断したばあい。

## Cloud Spanner

### 特徴

- RDBの中では唯一Writeのスケールアウトができる。
- アクセスポイントを複数リージョンに分散できる。
- ミリ秒単位のレイテンシ
- Google Cloudで独自のベクトル検索機能を実装している。
  - 厳密な近傍検検索をインデックスとして利用できる。
- SQLからVertexAIへのText Embedding APIを実行できる。

### ユースケース

- Cloud SQLやAlloyDBでは書き込み限界に達する規模のベクトル検索のユースケースがあるばあい。
- Cloud Spanner上に存在するデータに対してリアルタイム性の高いベクトル検索を実装したいばあい。
  - 商品情報テーブルの商品説明に対してあいまい検索する。
  - イベント情報テーブルのイベント説明に対してあいまい検索する。

## BigQuery

### 特徴

- アクセスポイントを複数リージョンに分散できる。
  - Writeのスケールアウトはできない。
- 大量データの集計を高速に実行できる。
- データベースのメンテナンスが不要。
- Google Cloudのさまざまなデータストアに対してクエリを実行できる。
- 秒単位のレイテンシ
- Google Cloudで独自のベクトル検索機能を実装している。
  - 近似最近傍探索をインデックスとして利用できる。
    - pgvectorではANNはメンテナンスが必要なものの、BigQueryでは自動でメンテナンスされる。
- SQLからVertexAIへのText Embedding APIを実行できる。

### ユースケース

- データベースのメンテナンス不要でベクトル検索を利用したいばあい。
- さまざまなデータソースのデータに対して単一のインタフェースでベクトル検索を利用したいばあい。
  - Bigtable上のIoT機器のログをベクトル化し、故障した機器の直近のログを検索し故障しそうな機器を検索する。
- 画像や動画に対するベクトル検索をしたいばあい。
  - BigLakeを利用してGCSと統合できるため。
- Cloud Loggingからシンクしたログに対してベクトル検索をしたいばあい。
  - ユーザーの行動ログなどをベクトル化し、不信な操作をするユーザーを発見したときに類似した行動をするユーザーを検索する。
- テーブルどうしをあいまい条件で結合して集計したいばあい。
  - 新規ユーザーの購買履歴と既存ユーザーの初期の購買履歴を結合し、将来のライフタイムバリューを計算する。

## データベースの特徴のまとめ
ここまで紹介したデータベースごとのベクトル検索の特徴をまとめたものが以下です。
![データベースの特徴まとめ](/images/compare_vector_searches_on_google_clouds_rdb/comparison_matrix.png)

# まとめ

ここまで紹介したように、Google CloudではRDBでベクトル検索を利用する場合に採用できる多様な選択肢があります。
特にMySQL互換のものはOSS版でサポートされておらず、これといったプラグインも存在していないため特定のDBaaSでしかサポートされていない機能でもあります。

またこれまではベクトル検索の技術は技術力に優れた一部の企業でしか活用が難しかったですが、Google CloudのデータベースではSQLからVertex AI上の機械学習モデルをサポートしている製品も多く、他クラウドやOSSのセルフホストでは実現が難しいハイレベルなベクトル検索体験を実現できます。
個人的な意見になりますが仕事で複数のクラウドでデータベースやデータ分析関連の機能を触っている人間からすると、Google Cloudはデータ分析と機械学習関連は機能とプラットフォーム完成度が抜きん出ていると考えています。
一方でそれは他クラウドの独自機能にもいえますが、クラウドへのロックインの原因となるので必ずしもよいことばかりとも言えません。^[これはクラウドやサービスがなくなるといった部分の心配ではなく、コストが許容できないレベルになってしまう可能性を指しての内容となります。]

最後にGoogle Cloud上のどのRDBでベクトル検索を利用するべきかという問いへの回答としては、レイテンシが短くベンダーロックインを避けることができ、なによりも高機能なCloud SQL for PostgreSQLやメンテナンスが簡単でさまざまなデータストアのインタフェースを統合できるBigQueryを中心にワークロードの特性に応じてそのほかのデータベースを選択するとよいでしょう。

# 参考
<!-- ignore-start -->
- [Work with vector embeddings  |  Cloud SQL for MySQL  |  Google Cloud](https://cloud.google.com/sql/docs/mysql/work-with-vectors)
- [Work with vector embeddings  |  Cloud SQL for PostgreSQL  |  Google Cloud](https://cloud.google.com/sql/docs/postgres/work-with-vectors)
- [Work with vector embeddings  |  AlloyDB for PostgreSQL  |  Google Cloud](https://cloud.google.com/alloydb/docs/ai/work-with-embeddings)
- [Perform similarity vector search in Spanner by finding the K-nearest neighbors  |  Google Cloud](https://cloud.google.com/spanner/docs/find-k-nearest-neighbors)
- [Introduction to vector search  |  BigQuery  |  Google Cloud](https://cloud.google.com/bigquery/docs/vector-search-intro)
- [Manage vector indexes  |  BigQuery  |  Google Cloud](https://cloud.google.com/bigquery/docs/vector-index)
- [BigQuery のベクトル検索のご紹介 | Google Cloud 公式ブログ](https://cloud.google.com/blog/ja/products/data-analytics/introducing-new-vector-search-capabilities-in-bigquery)
- [Find anything blazingly fast with Google's vector search technology | Google Cloud Blog](https://cloud.google.com/blog/topics/developers-practitioners/find-anything-blazingly-fast-googles-vector-search-technology)
- [pgvector/pgvector: Open-source vector similarity search for Postgres](https://github.com/pgvector/pgvector)
- [協調フィルタリングとベクトル検索エンジンを利用した商品推薦精度改善の試み | メルカリエンジニアリング](https://engineering.mercari.com/blog/entry/20230612-cf-similar-item/)
- [OSS 分散近似近傍密ベクトル検索エンジンVald～導入と活用事例～ - Yahoo! JAPAN Tech Blog](https://techblog.yahoo.co.jp/entry/2023030730416279/)
- [Introducing Natural Language Search for Podcast Episodes - Spotify Engineering : Spotify Engineering](https://engineering.atspotify.com/2022/03/introducing-natural-language-search-for-podcast-episodes/)
- [Understanding the SCaNN index in AlloyDB | Google Cloud Blog](https://cloud.google.com/blog/products/databases/understanding-the-scann-index-in-alloydb?hl=en)
<!-- ignore-end -->

