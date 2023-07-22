---
title: "Terraformでmapにkeyが含まれないときにスキップしたい"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform"]
published: true
---

Google CloudではPublic IPを利用した際に割り振られる可能性のあるCIDRの一覧が[cloud.json](https://www.gstatic.com/ipranges/cloud.json)でJSON形式で公開されています。

この記事は雑な検証用のTerraformで承認済みネットワークにasia-notheast1のCIDRを全部登録してやろうとしたとき、上記のJSONファイルから`scope`がasia-northeast1の`prefixes.ipv4Prefix`を抜きだそうとしたときにハマったのでその対応方法のメモです

## 結論
以下のような感じで書いたら対応できました。
```hcl
contains(keys(hoge), "fuga") # hogeのkeyにhugaを含む場合True
```

## 現象
[cloud.json](https://www.gstatic.com/ipranges/cloud.json)を見ると分かるとおり、`prefixes`内のmapは`ipv4Prefix`をKeyとして持つ場合と`ipv6Prefix`をKeyとして持つ場合があります。(それ以外は全て共通)

そのような時に以下のようなTerraformを書くと`This object does not have an attribute named "ipv4Prefix".`というエラーを吐いて終了します。
```hcl
data "http" "google-cloud-public-cidr" {
  url = "https://www.gstatic.com/ipranges/cloud.json"
}
locals {
  cidr_blocks = [
    for prefix in jsondecode(data.http.google-cloud-public-cidr.response_body).prefixes :
    prefix.ipv4Prefix if prefix.scope == "asia-northeast1"
  ]
}
```

そのため`ipv4Prefix`が存在しない場合(`ipv6Prefix`が対象の場合)、そのmapオブジェクトをスキップする必要があります。

## 先行事例
なにか良い方法はないか？ と探したところ[Terraformでmap内の未定義keyを判定する](https://qiita.com/shigure_onishi/items/3d12062df93bf690764e)など未定義Keyを判別する方法は見付けたものの、今回解決したい問題には適応出来ませんでした。

## 解決方法
Terraformには`contains(list, string)`という`list`に`string`が含まれるかを調べる関数と、`keys(map)`という`map`が持つkeyの一覧を取得する関数を組み合わせたら良い感じに実現できました。

```
data "http" "google-cloud-public-cidr" {
  url = "https://www.gstatic.com/ipranges/cloud.json"
}
locals {
  cidr_blocks = [
    for prefix in jsondecode(data.http.google-cloud-public-cidr.response_body).prefixes :
    prefix.ipv4Prefix if prefix.scope == "asia-northeast1" && contains(keys(prefix), "ipv4Prefix") # この部分 
  ]
}
```

## まとめ
大体公式が何とかしてくれるからこんな素人が書いた資料なんか探してる暇あったら、おとなしく公式ドキュメント読んでください。
貴重な土曜日のn時間を無駄にしてしまった……
