---
title: "SQL*Loaderで複数の文字コードが混ざったデータをロードする"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oracle","encoding"]
published: true
---

# SQL*Loaderで複数の文字コードが混ざったデータをロードする
## 概要
単一のテキストファイル内で特定のカラムのみ文字コードが違うファイルをSQL\*Loaderでデータベースに取り込む方法

## 注意
本記事で扱っている対処方法はおそらく紛れ込んだ文字コードが

本来あるべき文字コードの一部として解釈できない場合使用できないと思います。(未検証)
最低限文字化けしながらも読み込める状態を想定しています。

## 結論
コントロールファイル内で文字コードの変換が必要なカラムに以下の関数を適用する。
```
column "CONVERT(:column, 'target_charset', 'src_charset'),"
```

## 経緯
先日仕事でAdobe Analyticsから出力されたデータを扱う機会がありました。

該当のデータは基本的にISO8759-1で出力されるようなのですが、一部カラム(evar1-evar255)はユーザー側で
設定するデータのためISO8859-1の文字コードでない場合があるそうです。

## 読み込み方法
Oracle DatabaseのSQL\*Loaderにはもともとデータベースの`CHARCTERSET`とファイルのエンコーディングが
異なっている場合、ファイルを読み込むときのSQL\*Loaderのコントロールファイル内で
`CHARACTERSET`を指定することで変換をすることができます。^[今回の解決方法が使えることから考えるにDBへのロード時に変換が行われているように見える]

しかし`CHARACTERSET`を指定する方法は単一文字コードで構成されていることを前提としているため、
複数の文字コードが混在したケースを処理することはできません。

そこでSQL\*LoaderのSQL演算子をフィールドへ適用する機能を用いることで回避ができます。
Oracle Databaseには`convert`という任意の文字コードから他の文字コードに変換する機能が提供されているため、
対象となるフィールドに`convert`を適用することで無理やりファイルの文字コードを統一することが可能です。
```
column "CONVERT(:column, 'target_charset', 'src_charset'),"
```
![](/images/load_complex_characterset_oracle_image1.png)

## まとめ
ファイル内の文字コードは統一しよう。
各種プログラミング言語で読み込みするときって対処法あるんですか？

## 参考
- Adobe Analytics[データフィードの内容 - 概要](https://experienceleague.adobe.com/docs/analytics/export/analytics-data-feed/data-feed-contents/datafeeds-contents.html?lang=ja#%E3%83%92%E3%83%83%E3%83%88%E3%83%87%E3%83%BC%E3%82%BF%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB)
- Adobe Analytics[データ列リファレンス](https://experienceleague.adobe.com/docs/analytics/export/analytics-data-feed/data-feed-contents/datafeeds-reference.html?lang=ja)
- SQL\*Loader制御ファイル・リファレンス [入力文字の変換](https://docs.oracle.com/cd/E96517_01/sutil/oracle-sql-loader-control-file-contents.html#GUID-5E050EBB-C889-48BE-8A30-524B20864317)
- SQL\*Loader制御ファイル・リファレンス [フィールドへのSQL演算子の適用](https://docs.oracle.com/cd/E96517_01/sutil/oracle-sql-loader-field-list-contents.html#GUID-83FF6EDC-C7F8-4F29-8994-59153BE31924)
