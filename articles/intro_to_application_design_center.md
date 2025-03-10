---
title: "Google CloudのTerraform職人が失職する機能が出てしまった……"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud"]
published_at: 2023-12-11 08:30
---

Google CloudがApplication Design Centerという、構成図を書けばTerraformを書いて、デプロイまで行う機能をリリースしました。^[2025年3月11日現在Previewステータス]

https://cloud.google.com/application-design-center/docs/overview 

どうやらGoogle CloudはTerraform職人を失職に追い込みたいようです。

# Application Design Centerの概要

> アプリケーション デザイン センターは、Google Cloud アプリケーション インフラストラクチャの設計、共有、デプロイに役立ちます。アプリケーション デザイン センターを使用すると、アプリケーション テンプレートを設計し、開発者と共有し、インスタンスをデプロイして、設計を進化させることができます。

つまりApplication Design Centerはインフラ構成図の設計・共有・デプロイを補助するツールで、アプリケーション開発者やプラットフォーム管理者が効率的に管理・構築していくことを目的として作られたツールです。

Application Design Centerの機能として、作成したインフラストラクチャのテンプレートをTerraformとして吐き出す機能もあります。

## Application Desing Centerのコアコンポーネント

Application Design Centerには下記の4コンポーネントがあります。

1. Spaces
2. Templates
3. Applications
4. Catalogs

ここではこれらのコンポーネントについて簡単な説明をします。

### Spaces

SpaceはApplication Design Centerのトップレベルのリソースです。すべてのリソースはSpaceに所属しています。

Space間は基本的に独立しています。Spaceを作成して開発チームに払い出すことで、開発チームがApplicationを自由に構成できます。

### Templates

Templateはインフラストラクチャの構成図を書くことで、ApplicaitionのTemplateを作成するツールです。

Google Clodで利用可能な機能はほとんど^[すべてかは確認していませんが主要なコンポーネントはサポートしているようです。]サポートしており、パラメータなどもほとんど設定可能です。

### Applications

ApplicationはTemplateをデプロイする単位で、同じ構成のApplicationを複数デプロイできます。

ApplicationはCode Build経由で、Terraformを利用して作成されます。

またここで作成されたApplicationはApp Hubで管理されます。

[https://cloud.google.com/products/app-hub?hl=ja](https://cloud.google.com/products/app-hub?hl=ja)

### Catalogs

CatalogsはTemplateを管理するツールです。TemplateをCatalogに追加することで、バージョン管理をできます。またSpace間でTemplateを共有できます。

Templateには下記の2種類があります。

1. Googleのテンプレート
2. 共有テンプレート

 「Googleのテンプレート」は三層構造アプリケーションや生成AIアプリケーションなどよくある構成のテンプレート集で、非エンジニアでも簡単にAppliationをデプロイできます。

「共有テンプレート」はユーザーがSpace間で共有するTemplateです。自由な設定のTemplateを共有できるので、プラットフォームの標準化などに役立ちます。

またTemplateは必要に応じてカスタマイズできます。

# Application Design Centerを試す

## Application Design Centerの初回セットアップをする

Application Design CenterはPreviewステータスなため、コンソールで検索することでのみ到達できます。

![](/images/intro_to_application_design_center/intro_to_application_design_center_01.png)

Application Design Centerを開くと、初回はスペースの作成と必要なAPIの有効化が求められるます。

任意のスペース名を入力し、Complete Setupを押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_02.png)

数分ほどするとスペースの作成とAPIの有効化が完了し、下記のような画面になります。

これで最低限のセットアップは完了です。

![](/images/intro_to_application_design_center/intro_to_application_design_center_03.png)

## Templateを作成する

続けてTemplateを作成します。

画面左のナビゲーションバーから「テンプレート」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_04.png)

するとTemplateの管理画面に移動します。この時点ではTemplateを作成していないため、何も表示されていません。

続いて画面丈夫の「Create Template」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_05.png)

押下するとTemplateの設定画面に遷移します。

ここで、下記の入力を求められますが入力内容はシンプルなため説明を割愛します。

- Template name
- Display name (Optional)
- Description (Optional)
- Attributes
    - **Environment**
        - PRODUCTION
        - STAGING
        - DEVELOPMENT
        - TEST
    - **Criticality**
        - HIGH
        - MEDIUM
        - LOW
- Business Owner (Optional)
- Developer Owners (Optional)
- Operator Owners (Optional)

上記内容を入力して画面下部の「Create Template」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_06.png)

すると、テンプレートに配置するコンポーネントやコンポーネント間の関係を編集する画面に遷移します。

今回は画面左のコンポーネント一覧から「Cloud Run」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_07.png)

Cloud Runのコンポーネントが画面中央の描画スペースに移動され、画面右ではCloud Runのパラメータなどが設定できます。この時選べるRegionは数リージョンに限られているものの、ほとんどのパラメータはサポートされていそうです。

必要な設定をしたら、画面右したの「保存」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_08.png)

すると下記のようにコンポーネントに保存した設定が反映されます。

ここでコンポーネントの右上にある「+」を押下すると、接続可能なコンポーネントの一覧が表示されます。

![](/images/intro_to_application_design_center/intro_to_application_design_center_09.png)

![](/images/intro_to_application_design_center/intro_to_application_design_center_10.png)

Secret Managerを選択すると、下記のようにSecret ManagerとCloud Runが結線で結ばれた状態として追加されます。

Cloud Runと同様に必要な設定をして「保存」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_11.png)

これらの作業を数回繰り返すことで、下記のようなよくある三層構造のアプリケーションのテンプレートが作成できます。

ここで「Add to Catalog」を押下するとTemplateをCatalogに登録できます。複数回押下すると複数のRevisionを保存できます。

![](/images/intro_to_application_design_center/intro_to_application_design_center_12.png)

![](/images/intro_to_application_design_center/intro_to_application_design_center_13.png)

他にはDesign横のコードを押下するとCloud Shellで対応するTerraformコードを見ることができたり、「コードを取得」を押下するとZip形式でTerraformコードをダウンロードできます。

この時「コードを取得」は常にエラーメッセージを出力しますが、ダウンロードには問題ありません。

実際に出力されるコードは下記のリポジトリのものです。

https://github.com/nnaka2992/application-design-center-poc/tree/main

最後に画面右上の「Configure an App」を押下するとTemplateに紐づくApplicationの一覧や、新規の「Create new application」が表示されるので、それぞれ押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_14.png)

するとデプロイするApplicationの設定画面が表示されます。ほとんどの項目はTemplateと共通です。

- Application name
- Display name (Optional)
- Description (Optional)
- Region
    - いくつかのRegionからドロップダウンで選択
- Input Attributes
    - Application scope
        - REGIONAL
        - ZONAL
    - **Environment**
        - PRODUCTION
        - STAGING
        - DEVELOPMENT
        - TEST
    - **Criticality**
        - HIGH
        - MEDIUM
        - LOW
- Business Owner (Optional)
- Developer Owners (Optional)
- Operator Owners (Optional)

それぞれ必要な設定をしたら、「Craete Application」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_15.png)

ここまで設定するとApplicationのリソースが作成されます。

### Applicationをデプロイする

「Create Application」を押下するとTemplate通りの構成のApplicationsのリソースが作成され、デプロイが可能になります。

デプロイするにはまず画面右上の「デプロイ」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_16.png)

デプロイを押下するとデプロイ時に利用されるサービスアカウントの指定画面が表示されます。このサービスアカウントはデプロイ時にしか利用されない様です。

> Select the service account and review your application before deployment. This service account will only be used in deploying your application.

利用するサービスアカウントに合わせた設定し、画面中央下部の「デプロイ」を押下します。

![](/images/intro_to_application_design_center/intro_to_application_design_center_17.png)

各設定が正しく行えている場合、しばらく待つとApplicationが作成されます。

下記はエラー発生時の画面で、「View Logs」を押下をするとCode Buildのログが表示されます。

![](/images/intro_to_application_design_center/intro_to_application_design_center_18.png)

Code Buildのログの内容を見るに、TerraformをCode Buildで実行されているようです。

この時作成に失敗したリソースは削除されないようです。初期作成では削除されてものの、追加の開発であればリソースの破壊が発生しかねないので、削除されないのが妥当です。

ちなみに今回のエラーの原因はあやまったデータベースのインスタンスタイプを指定下からでした。Enterprise Plusでのみ利用できるインスタンスタイプをEnterpriseで利用しようとしていたためです。

![](/images/intro_to_application_design_center/intro_to_application_design_center_19.png)

作成したアプリケーションのリビジョンを変更はできないので今回は一削除してから再作性します。

スリードットから「アプリケーションの削除」が可能です。

![](/images/intro_to_application_design_center/intro_to_application_design_center_20.png)

![](/images/intro_to_application_design_center/intro_to_application_design_center_21.png)

このとき、Applicationのリソースを削除する作成されたリソースも自動的に削除されるため、運用を開始しているコンポーネントについては注意が必要です。

> This application and all resources created when it was configured and deployed will be deleted. This application will no longer be available from the Applications page.
> 
> ![](/images/intro_to_application_design_center/intro_to_application_design_center_22.png)

下記はデプロイが成功した画面で、「View app in App Hub」を押下するとApp Hubからアプリケーションを管理できます。

![](/images/intro_to_application_design_center/intro_to_application_design_center_23.png)

![](/images/intro_to_application_design_center/intro_to_application_design_center_24.png)

# まとめ

Application Design Centerを利用すると構成図を書くだけでインフラストラクチャーの構成が可能になり、またそれを環境ごとに作成できます。

これを手動で行おうとすると、リソース数によっては非常にたいへんになります。そのためTerraformといったIaCツールに慣れていない場合、大きな手間となります。

Application Design Centerを利用するとそういった手間を大幅に削減でき、「Googleのテンプレート」を利用するとGoogle Cloudに詳しくない人でも、生成AIアプリケーションや基本的な構成を作成できるという意味でも非常に有用なツールです。

一方で今までTerraformを利用していた組織が乗り換えるほどかと言われると、そんなことはありませんでした。ノーコードツールの宿命ですが、Terraformを直接記載するほどの表現力はないため、本番環境ではインスタンスを冗長化するけれど、開発環境ではしないといった設定が困難です。

一方でPoC環境をサクッと作りたい場合や、Terraformコードのひな型を作成したい場合には一定のメリットがあるでしょう。
