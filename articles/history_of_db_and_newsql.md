---
title: "DBMSの歴史とNewSQL"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NewSQL", "DB"]
published: true
---
この記事はDBMSの登場以前から現代のDBMSを取り巻く環境までを振り返ることで、
なぜNewSQLが必要とされ登場したのかをまとめます。

## おことわり
筆者はあくまでDBMSユーザーであり、研究者ではないため内容は個人の見解です。
また対象読者はある程度DBMSに関わりがあり、OLTPやOLAP、列指向や行指向といった
基本的な単語を理解しているものとします。
またNewSQLの技術的詳細はスコープ外とします。

## DBMS以前
データベースという言葉は1950年代に米軍が情報基地を集約したことに由来します。
一方で学術的なデータベースの起源はW. C. McGeeが1959年に発表した[論文](https://dl.acm.org/doi/10.1145/320954.320955)が
始まりとされています。

当時アメリカが世界一の科学力を自称していたにも関わらず、1957年にソ連が人類初の人工衛星の
打ち上げに成功した[スプートニク・ショック](https://ja.wikipedia.org/wiki/スプートニク・ショック)を受け、
安全保証上の理由から一箇所に集中していたコンピュータとそのネットワークを分散させる必要が発生しました。

それにともない点在する拠点間で膨大なデータを収集、処理する必要がありました。

## DBMSの登場とRDBMS
1966年、IBMが先述のW. C. McGeeが提案したデータベースである階層型データベースの商用実装である[IMS](https://www.ibm.com/products/ims)を発表しました。IMSはRDBMSのようにトランザクションを管理出来るDBMSでした。
階層型データベースは木構造でデータを表わす方式をとっていました。

その3年後の1969年、CODASYL(Committie on Data Systems Languages)によりネットワークモデルのDBMSを発表しました。
このデータモデルは階層型データベースが抱えていた、木構造の子要素が複数の親要素を持てないという欠点を克服したものでした。

しかしいずれのDBMSもデータモデルに柔軟性が乏しく、データの物理的独立性を達成出来ず、そしてrecord-at-a-time形式のインターフェイスのみを提供していたことからデータ問い合わせに課題がありました。

このような背景から1970年、Ted Coddはリレーショナルモデルを採用したリレーショナルデータベース(RDB)を提案しました。
このデータモデルではIMSやCODASYLが抱えていた問題を解決したことに加え、マイコンブームで大量に供給された新しいアーキテクチャのPCで利用出来るOracle(1979)やINGRESS(1980頃)が登場しました。
その後、当時のDBMSの中心であったメインフレーム向けにもIBMが[Db2](https://www.ibm.com/jp-ja/products/db2)をリリースしたことで、覇権となるデータモデルは階層型・ネットワークモデルからRDBに変化していきました。

## Webの時代
RDBMSの登場から数年後にインターネット通信の一般利用が開始され、2000年頃にはそれまで以上に大量のデータが生成されるようになりました。

### NoSQL
スキーマレスな自己記述型データであるXMLが登場し、複雑なネットワーク指向データモデルとしてXMLデータベースが研究されるようになりました。これがRDB時代以降のNoSQLの走りでした。

その後も大量に生成されるRDBでは対応しきれないWriteヘビーなワークロードに適応すべく、高可用性と高いスケーラビリティを目的として、ACIDトランザクションやSQLのかわりに個別のAPIを使用してデータ問い合わせを行なうOSSのNoSQL系DBMSが次々と登場しました。

### ビッグデータとデータ分析
大量のデータが生成されたことの帰結として、それらのデータを利用しようという流れが発生しました。

大量のデータを処理するために分散したシェアードナッシング方式のRDB/SQLベースの商用データウェアハウスが登場しました。
これらデータウェアハウスは、従来のRDBとは異なりデータの圧縮効率や検索効率、集計処理など分析ワークロードに優れた行指向ストレージを使用していました。

DWHの登場と同時期に分散プログラミングによる大規模データの分析手法としてGoogleが[MapReduce](https://static.googleusercontent.com/media/research.google.com/ja//archive/mapreduce-osdi04.pdf)を提案しました。
論文としてアイデアは公開されていたもののGoogleによるソフトウェア実装は未公開であった為、その後Yahoo!がHadoopとして実装しOSSで公開しました。

## NoSQLからNewSQLへ
[Bigtable](https://cloud.google.com/bigtable?hl=ja)や[Cassandra](https://cassandra.apache.org/_/index.html)といったトランザクションや一貫性を一部犠牲にしたNoSQLにより、Webアプリケーションなどから発生する大量の書込み処理にも対応出来るようになりました。

一方で複雑なスキーマを持つアプリケーションの開発はNoSQLのスキーマレスな特性では自由度が多きすぎ、またトランザクションや強い一貫性を必要とするアプリケーションではNoSQLは機能が不足していました。

そこで2013年にGoogleが登場したのがグローバルに分散するRDBMSとしてSpannerを発表しました。これが現在主流のNewSQLデータベースの源流です。
Spanner以前にも[VoltDB](https://www.voltactivedata.com/)や[ClustrixDB](https://mariadb.com/ja/resources/blog/scale-out-database-clustrixdb/)などNewSQLと呼ばれるデータベースがあったようですが、Spanner以前と以降でアーキテクチャが大きく変ります。
またNewSQL以前からVitessやCitusといったシャーディングによりRDBMSを水平分割し、スケーリングする方法は提供されていました。
しかしシャーディングによる分割はシャーディングを前提としたデータモデリングや正しいシャードにルーティングするためのロジック、シャードを跨いだジョインやトランザクションロジックの開発など様々な障害がありました。

NewSQLはNoSQLとシャーディングしたRDBMSの抱える問題を解決し、一貫性と可用性をあきらめることなく分断耐性を実現しスケールアウトによる書込みのスケールを可能としました。

## NewSQL and Beyond
近年のクラウドコンピューティングやKubernatesなどによりGoogleのような世界中にデータセンタを持つ一部の大企業以外でも、NewSQLの利用が一般的になり[TiDB](https://docs.pingcap.com/ja/tidb/v5.4)や[CockroachDB](https://www.cockroachlabs.com/)、[YugabyteDB](https://www.yugabyte.com/)など様々なSpannerクローンと言えるNewSQLも登場しました。

そのため競争が加速し様々な機能が追加されるようになりました。
とくにTiDBの発展は目覚ましく、TiFlashと呼ばれる行指向エンジンによる分析向けのデータアクセスにも対応しています。
こういったOLTPにもOLAPにも対応したワークロードはHTAPと呼ばれ、MySQL HeatwaveやAlloyDB for PostgreSQLなどNewSQL以外でも用いられる最先端のアーキテクチャです。

またCockroachDBやYugabyteDBはCassandra互換のAPIを提供し、従来のNoSQLワークロードのサポートをしています。

このような様々なインターフェースやNoSQL機能への対応は従来のRDBMSでも進んでおり、そういった優れたRDBMSに対抗するためにNewSQLでも様々な機能が追加されて行くでしょう。

## 参考
- 辻靖彦・芝崎順司・浅井紀久夫・小林亜樹 (2017). データベース 一般社団法人教育振興会
- [NoSQL、NewSQL、そしてその先](https://www.infoq.com/jp/news/2011/04/newsql/)
- [2020年現在のNewSQLについて](https://qiita.com/tzkoba/items/5316c6eac66510233115)
- [What Goes Around Comes Around](https://15721.courses.cs.cmu.edu/spring2023/papers/01-intro/whatgoesaround-stonebraker.pdf)
- [History of Databases](https://15721.courses.cs.cmu.edu/spring2023/slides/01-history.pdf)
- [Why sharding is bad for business](https://www.cockroachlabs.com/blog/sharding-bad-business/)
- [Spanner: Google’s Globally Distributed Database](https://dl.acm.org/doi/10.1145/2491245)
