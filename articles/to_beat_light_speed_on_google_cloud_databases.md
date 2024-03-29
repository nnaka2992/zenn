---
title: "光に負けルナ~Google Cloudでのマルチリージョンデータベースについて~"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Google Cloud", "Database"]
published: true
---

クラウドを利用する一番のメリットの一つとしてオンデマンドでリソースを調達し、アクセス負荷に応じてスケールイン・アウト出来ることが上げられます。
そのため大体のアプリケーションではシングルリージョンまたは隣接するリージョン2~3程度で運用を始めることが多いと思います。
(日本の場合asia-northeast-1とasia-northeast-2など)
アプリケーションがグローバルに拡大すると、それだけ物理的な距離が広がりユーザ・サーバ間のアクセスにかかる時間が拡大します。
例えばユーザ・サーバ共に日本にある場合(沖縄・北海道間約3,000km)、ネットワークによる遅延は片道約15ms以下で収まり往復でも30ms程度になる。
一方でユーザ・サーバがそれぞれサンフランシスコと東京にある場合(約10,000km)ネットワークに寄る遅延は往復で約100ms程度になります。[^1]

Amazonによれば100msの遅延が発生すると1%のユーザが離脱する[^2]ため、地理的距離に起因するレイテンシは簡単に受入れられるものではありません。
そういった問題を解決するためにアプリケーション開発者はCDNを利用したり、ユーザがアクセスするエンドポイントを最も近い場所にルーティングするなどして対応しています。

基本的にモダンなアプリケーションはステートレスにして、スケールアウト出来るように設計することが多いため複数リージョンに跨るアプリケーションにも低レイテンシで対応することが出来ます。

一方でデータベースはステートフルなため、アプリケーションほど柔軟に地理的分散に対応することが難しいです。
この記事ではGoogle Cloudではどのようにレイテンシを最小化するか考えます。

## Google Cloudで利用できるマネージド・データベースの復習
地理的分散とレイテンシの最小化を考えるにあたり、まずはGoogle Cloudで利用可能なマネージド・データベースを整理します。
Google Cloudが公式でサポートするデータベースは勿論、一部マーケットプレースで提供しているデータベースも扱います。

1. Cloud SQL / AlloyDB for PostgreSQL
   Google Cloudのデータストアで最も利用されるデータベースサービスです。
   AlloyDB for PostgreSQLも地理的分散が不得意なため、ここではひとまとめにしています。
   この記事ではこれ以上は触れません。
1. Cloud Spanner
   Google Cloudが提供するデータストアのなかで唯一地理的分散を前提としたトランザクションをサポートしたRDBです。
1. Yugabyte Anywhere
   Spannerと同じく地理的分散とマルチライターを前提としたNewSQLであるYugabyteDBです。
1. CockroachDB Dedicated
   Spannerと同じく地理的分散とマルチライターを前提としたNewSQLであるCockroachDBです。
1. Cloud Bigtable
   ワイドカラムストアと呼ばれるデータストアで、トランザクションをサポートしません。
   しかし地理的分散は可能なソリューションです。
1. Firestore
   Spannerと同じくトランザクションをサポートし、かつ地理的分散を可能とするデータベースです。
   ただしSpannerと異りFirestoreはドキュメントDBです。
1. MongoDB Atlas
   Firestoreと同じドキュメントDBの一種です。
   ただしトランザクションはサポートしていません。

## トランザクションが不要な場合
マルチリージョン・ライターデータベースについて考える場合、一番に考慮すべきはトランザクションの必要性です。
トランザクションをサポートしたデータベースでもマルチリージョン・ライターを実現することは出来ますが、その分パフォーマンスをある程度犠牲にする必要があります。
所謂NoSQLと言われるデータベースの殆どはトランザクションをサポートしないことで、マルチライターを実現しています。
BigtableもMongoDB Atlasもデータを複数のリージョンに分散したノードにシャーディングするアプローチと、結果整合性を採用することで実現しています。

これらのタイプのデータベースはトランザクション制御を必要としないため、高速なレスポンスを必要とするユースケースやパフォーマンスを犠牲にせず
に地理的分散を実現したい場合に向いています。

## トランザクションが必要な場合
### SQLインターフェースが不要な(RDBでなくてもいい)場合
FirestoreはドキュメントDB/NoSQLの中でも比較的珍しいACIDトランザクションを完全にサポートしたデータベースです。
またマルチリージョン・ライターをサポートしているため、地理的分散を行なうことも出来ます。

FirestoreはNoSQLの手軽さとトランザクションの堅牢さ、NewSQLのスケール能力の良いとこ取りをしたデータベースでモバイルアプリケーションやwebアプリケーションを中心に様々な用途に利用することが出来ます。
とはいいつつ私自身非RDBはあまり専門でやっていないため、ユースケースをとらえきれていない部分があります。

### SQLインターフェースが必要な場合
マルチリージョン・ライターが求められ、トランザクションが必要かつSQLインターフェースが必要な場合に力を発揮するのがCloud Spannerを始めとしたNewSQLです。
SpannerやYugabyteDB、CockroachDBと同じNewSQLであるTiDBは、マルチリージョンが非推奨なためここでは扱いません。

いままで紹介したBigtableやMongoDB、Firestoreと違いRDBであることが大きな挑戦を産みます。

NewSQLはRDBであるため外部キーが指すレコードが異なるリージョンに存在することがありえます。
前述のとおりリージョンを横断するアクセスはネットワーク起因のレイテンシが大きくなります。
TiDBを除くNewSQLでは地理的分散を前提としているため、そういった場合へのソリューションとしてSpannerであればinterleaveをYugabyteDBであればCollocationと言われる特定のデータを同じリージョンに配置する手法によって解決しています。
CockroachDBでは以前はinterleaveとしてサポートしていたものの、パフォーマンス的に優位性がないとされ完全に削除されました。

このようなRDBだからこそ課題になる異なるリージョンに対して外部キーを参照することを制御することが出来ます。
またここで触れたマルチリージョンによる課題は殆どの場合Writeに対しての問題で、ReadについてはRaft/Paxosグループによって実現されるレプリケーションによって解決されています。

## まとめ
長々とGoogle Cloudでデータベースを分散する方法について書いてきましたが、レイテンシや管理コスト的に出来る限り分散は避けた方がいいです。

[^1]: Cat-6Aを前提とした理論値
[^2]: [Marissa Mayer at Web 2.0](http://glinden.blogspot.com/2006/11/marissa-mayer-at-web-20.html)
