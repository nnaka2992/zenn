---
title: "AlloyDB omni on Kubernetesを眺める"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DB","Kubernetes","GCP","PostgreSQL"]
published: true
published_at: 2023-12-06 08:30

---
## 背景を眺める
2023年10月12日にAlloyDB OmniのGAに併せてAlloyDB Omni on Kubernetesが発表されました。
https://cloud.google.com/blog/ja/products/databases/announcing-the-general-availability-of-alloydb-omni
これはAlloyDB OmniをKubernetes上で管理するOperatorをgcsで経由で配布しHelmを利用してインストールさせています。
今回はこのAlloyDB Omni on Kubernetesのチュートリアルを行ない、そこでは触れられていないOperatorの詳細を眺めます。
結論から言うとPreview版ということもあり、AlloyDB Cluster作成以外の操作が出来そうにないため本格的な利用は難しいでしょう。

## 環境を眺める
今回はUbuntu上で動作するkind上でAlloyDB omni on Kubernetesを試します。
```bash
> cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.3 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy

> kind --version
kind version 0.20.0

> docker --version
Docker version 24.0.7, build afdd53b

> kubectl version
Client Version: v1.28.3
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.27.3

> helm version
version.BuildInfo{Version:"v3.13.2", GitCommit:"2a2fb3b98829f1e0be6fb18af2f6599e0f4e8243", GitTreeState:"clean", GoVersion:"go1.20.10"}
```

## ドキュメントを眺める
[公式ドキュメント](https://cloud.google.com/alloydb/docs/omni/deploy-kubernetes)に記載されている手順に従って動かしてみます。

### 事前準備
kindクラスターを作成します
```bash
> curl -O https://gist.githubusercontent.com/nnaka2992/6ff9794181a6f5d77d4cc3ab819c9366/raw/d8424130437c87f376774725bdbb1da898fce2a8/kind-alloydb-omni-config.yaml
> kind create cluster --config kind-alloydb-omni-config.yaml
```

kindにcert-managerを作成します
```bash
> kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml
```

### AlloyDB Omni Operatorをインストールする
環境変数を設定します
```bash
> export GCS_BUCKET=alloydb-omni-operator
> export HELM_PATH=$(gcloud storage cat gs://$GCS_BUCKET/latest)
> export OPERATOR_VERSION="${HELM_PATH%%/*}"
```

AlloyDB OmniのOperatorをCloud Storageからダウンロードします
```bash
> gcloud storage cp gs://$GCS_BUCKET/$HELM_PATH ./
```

AlloyDB OmniのOperatorをHelmを利用してインストールします
```bash
> helm install alloydbomni-operator alloydbomni-operator-${OPERATOR_VERSION}.tgz \
    --create-namespace\
    --namespace alloydb-omni-system\
    --atomic\
    --timeout 5m
NAME: alloydbomni-operator
LAST DEPLOYED: Sun Nov 12 23:47:00 2023
NAMESPACE: alloydb-omni-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

### AlloyDB Omni Clusterを作成する
以下のyamlを利用してAlloyDB omniのクラスタを生成します。
```bash
> curl -s https://gist.githubusercontent.com/nnaka2992/80ac78f7afa2462367b5c18242db3f6b/raw/ccb9fa051dfa4da418c41e1cd8918e53cce13be2/alloydb_omni.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-pw-alloydb-omni-poc
type: Opaque
data:
  alloydb-omni-poc: "dGVzdF9wYXNzd29yZAo=" # echo "test_password" | base64 --
---
apiVersion: alloydbomni.dbadmin.goog/v1
kind: DBCluster
metadata:
  name: alloydb-omni-poc
spec:
  primarySpec:
    adminUser:
      passwordRef:
        name: db-pw-alloydb-omni-poc
    version: "15"
    resources:
      cpu: 1
      memory: 8Gi
      disks:
      - name: DataDisk
        size: 10Gi

> kubectl apply -f https://gist.githubusercontent.com/nnaka2992/80ac78f7afa2462367b5c18242db3f6b/raw/ccb9fa051dfa4da418c41e1cd8918e53cce13be2/alloydb_omni.yaml 
secret/db-pw-alloydb-omni-poc created
dbcluster.alloydbomni.dbadmin.goog/alloydb-omni-poc creatde
```
この`apply`で以下のようなリソースが作成されます
```bash
> kubectl get all
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/al-b8d4-alloydb-omni-poc-0                          3/3     Running   0          32h
pod/al-b8d4-alloydb-omni-poc-monitor-7d64558d76-lwgkl   1/1     Running   0          32h

NAME                                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/al-b8d4-alloydb-omni-poc-dbd   ClusterIP   10.96.29.154    <none>        3203/TCP   32h
service/al-b8d4-alloydb-omni-poc-ilb   ClusterIP   10.96.210.150   <none>        5432/TCP   32h
service/kubernetes                     ClusterIP   10.96.0.1       <none>        443/TCP    32h

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/al-b8d4-alloydb-omni-poc-monitor   1/1     1            1           32h

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/al-b8d4-alloydb-omni-poc-monitor-7d64558d76   1         1         1       32h

NAME                                        READY   AGE
statefulset.apps/al-b8d4-alloydb-omni-poc   1/1     32h
```

Operatorのソースを見ることができないため一部想像を含みますが、作成されたリソースの関係は画像のとおりです。
![alloydbomni_operator drawio](https://gist.github.com/assets/66209003/d3de17ca-1276-474c-a061-6a13262d13b2)
リソースは大きく分けてAlloyDB本体とモニタリング(おそらくPrometheus？)の二つに分類できます。

1. AlloyDB
   StatefulSetからなるリソース郡です。
   StatefulSetのPodは以下のコンテナによって構成されています。
   - initContainer
     - dbinit
       後述のdatabaseコンテナが利用する起動スクリプトやコンフィグファイルなどを準備している様子でした。
   - container
     - database
       AlloyDB omni本体が動作するコンテナ
     - logrotate-agent
       詳細は不明ですが`kubectl logs al-b8d4-alloydb-omni-poc-0 logrotate-agent `
       を見る限り、`postgres`コマンドの実行ログの処理をしているよう様子でした。
     - memoryagent
       詳細は不明ですが`cgmon`というメモリの監視を行なうツールが動いている様子でした。
   
2. モニタリング
   Deploymentからなるリソースです。
   Deploymentは以下のコンテナを含むReplicaSetによって構成されています。
   - monitor
     AlloyDBのdatabaseコンテナにポート9187でアクセスし、メトリクスを収集しているようです。
     Annotationsにあるpromethus.ioの文字からprometheusのNode Exporterなどが動いていると想像できます。

### 動作確認
AlloyDBのpod上のpsqlクライアントからAlloyDBにアクセスしてみる。

```
> export DBPOD=`kubectl get pod -l alloydbomni.internal.dbadmin.gdc.goog/dbcluster=alloydb-omni-poc -l alloydbomni.internal.dbadmin.gdc.goog/task-type=database -o jsonpath='{.items[0].metadata.name}'`

> kubectl exec -ti $DBPOD -c database -- /bin/psql -h localhost -U postgres
Password for user postgres: 
psql (15.2)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_128_GCM_SHA256, compression: off)
Type "help" for help.

postgres=# SELECT count(*) from pg_settings;
 count 
-------
   458
(1 row)


postgres=# SELECT name, setting FROM pg_settings WHERE name like '%alloy%';
                     name                     |            setting             
----------------------------------------------+--------------------------------
 alloydb.allow_passwordless_local_connections | off
 alloydb.audit_log_line_prefix                | %m [%p]: [%l-1] db=%d,user=%u 
 alloydb.custom_reportv2                      | on
 alloydb.enable_pgaudit                       | off
 alloydb.extension_maintenance                | off
 alloydb.fs_access                            | off
 alloydb.iam_authentication                   | off
 alloydb.log_throttling_window                | 0
 alloydb.pg_authid_select_role                | 
 alloydb.pg_shadow_select_role                | 
 alloydb_prefetch_size_for_vacuum             | 1024
 alloydbg.coredump_at_location                | file_name:XX
 alloydbg.coredump_on_error                   | XXXXX
 alloydbg.enable_coredump_on_PANIC            | off
 enable_alloydb_vacuum_verbose_logs           | off
(15 rows)

postgres=# exit
```
PostgreSQL 15のruntimeパラメータは約380ほどなのに対して、ここでは458のパラメータがあります。
またruntimeパラメータ名にalloyを含むものがあることからもAlloyDBが起動していることがみてとれます。

## Operatorの中身を眺める
軽くチュートリアルを試しただけでは作成されるリソースが何なのかも、どんなオプションが存在するのかも分かりません。
なのでHelmの中身を見ていきます。

### alloydbomni-operator-${OPERATOR_VERSION}.tgzの構成
`alloydbomni-operator-${OPERATOR_VERSION}.tgz`の中は以下のようなHelmが保存されています。
これは`alloydbomni-operator-fleet`と`alloydbomni-operator-local`の2つのChartとそれを呼び出すChartから構成されていることが分かります。
```
./alloydbomni-operator
|--.helmignore
|--Chart.yaml
|--charts
|  |--alloydbomni-operator-fleet
|  |  |--.helmignore
|  |  |--Chart.yaml
|  |  |--crds
|  |  |  |--crds.yaml
|  |  |--templates
|  |  |  |--crs.yaml
|  |  |  |--webhooks.yaml
|  |  |--values.yaml
|  |--alloydbomni-operator-local
|  |  |--.helmignore
|  |  |--Chart.yaml
|  |  |--crds
|  |  |  |--crds.yaml
|  |  |--templates
|  |  |  |--crs.yaml
|  |  |  |--webhooks.yaml
|  |  |--values.yaml
|--values.yaml
```

以下はこのHelmによって作成されたCustomResourceDefinitionです。
これらはAPIVERSIONをベースに`alloydbomni.dbadmin.goog/v1`と`alloydbomni.internal.dbadmin.goog/v1`の2種類に大別することができ、
それぞれ`alloydbomni-operator-fleet`と`alloydbomni-operator-local`に当ります。
```
> kubectl api-resources | grep alloy
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
backupplans                       aoobp        alloydbomni.dbadmin.goog/v1            true         BackupPlan
backups                           aoob         alloydbomni.dbadmin.goog/v1            true         Backup
controlplaneagentsversions                     alloydbomni.dbadmin.goog/v1            true         ControlPlaneAgentsVersion
databaseversions                               alloydbomni.dbadmin.goog/v1            false        DatabaseVersion
dbclusters                        aood         alloydbomni.dbadmin.goog/v1            true         DBCluster
failovers                                      alloydbomni.dbadmin.goog/v1            true         Failover
restores                          aoor         alloydbomni.dbadmin.goog/v1            true         Restore
softwarelibraries                              alloydbomni.dbadmin.goog/v1            false        SoftwareLibrary
failovers                                      alloydbomni.internal.dbadmin.goog/v1   true         Failover
instancebackupplans               aooibp       alloydbomni.internal.dbadmin.goog/v1   true         InstanceBackupPlan
instancebackups                   aooib        alloydbomni.internal.dbadmin.goog/v1   true         InstanceBackup
instancerestores                  aooir        alloydbomni.internal.dbadmin.goog/v1   true         InstanceRestore
instances                         aooi         alloydbomni.internal.dbadmin.goog/v1   true         Instance
lrojobs                                        alloydbomni.internal.dbadmin.goog/v1   true         LROJob
```

### CRDの説明
前述のとおりCRDは`alloydbomni.dbadmin.goog/v1`と`alloydbomni.internal.dbadmin.goog/v1`によって構成されていますが、
crd.yamlを読む限り`alloydbomni.dbadmin.goog/v1`はユーザーが利用することを想定しています。
その一方で`alloydbomni.internal.dbadmin.goog/v1`は`alloydbomni.dbadmin.goog/v1`が内部的に利用することを想定しているのか、
yamlファイルのコンフィグとして利用することは出来なさそうなので説明を省略します。

#### apiversion: alloydbomni.dbadmin.goog/v1
- backupplans
  - description: BackupPlan is the Schema for the backupplans API
  - parameters
    - PITREnabled
      - PITRを有効化するかどうか
    - backupLocation
      - backupの保存先
      - GCSかS3か
      - 折角Omni on KubernetesにするならほかのObject StorageやVPCの選択肢が欲しかった。今後に期待
    - backupRetainDays
      - backupの保持期間
      - 1~90日
      - おそらくObject StorageのRetantionポリシーで制御している。要検証
    - cronSchedule
      - cron形式のバックアップスケジュール指定
    - dbclusterRef
      - バックアップ対象となるDBクラスター
    - paused
      - バックアップスケジュールの休止フラグ
    - physicalBackupSpec
      - フルバックアップを取得するか差分バックアップを取得するか
- backups
  - description: Backup is the Schema for the backups API
    - 実際に取得されたバックアップ？
  - parameters
    - backupPlanRef
      - このバックアップに紐付くbackupplan
    - dbclusterRef
      - このバックアップに紐付くDBクラスター
    - manual
      - バックアップがスケジュールで取得されたか手動で取得されたか
    - physicalBackupSpec
      - フルバックアップか差分バックアップか
- controlplaneagentsversions
  - description: ControlPlaneAgentsVersion specifies the new image path for each control plane agent component during the DBC provisioning and upgrade.
    - DBクラスターのプロビジョニング・アップグレード時に利用するコントロールプレーンエージェントのイメージパス
      - pgpool-IIやpatroniのようなもの？
  - parameters
    - components
      - コンポーネント名とそれに対応するURI
    - databaseengine
      - データベースエンジンの種類
      - PostgreSQL、Oracle、AlloyDBOmniが選択可能
        - Oracleとは 
    - version
      - control planが対応するエージェントスイートのバージョン
- databaseversions
  - description: DatabaseVersion is the Schema for the databaseversion API
  - parameters
    - databaseEngine
      - データベースエンジンの種類               
      - PostgreSQL、Oracle、AlloyDBOmniが選択可能
    - defaultMinorVersion
      - デフォルトのマイナーバージョン
    - majorVersion:
      - メジャーバージョン
    - minorDatabaseVersions
      - ODSビルド番号
      - components
        - コンポーネント名とそれに対応するURI
      - データベースエディション
        - Oracleを意識したパラメータ？
      - version
        - データベースのバージョン
      - versionDisplayName
        - ユーザー向けの表示名 
    - versionDisplayName
      - ユーザー向けの表示名
- dbclusters
  - description: DBCluster is the Schema for the dbclusters API
  - parameters
    - allowExternalIncomingTraffic
      - 外部からのTrafficを受け入れるかどうか
    - availability
      - HA構成にするかどうか
    - databaseImage
      - ユーザー指定のカスタマイズされたDatabaseイメージ
    - isDeleted
      - DBClusterがソフトデリートされたかどうか
    - mode
      - DBクラスターのモード
      - ""かdisasterRecoveryか
        - おそらくAlloyDB Omni単体で利用するかAlloyDBのDR先として利用するかの選択
    - primaryCluster
      - おそらくAlloyDBをマルチクラスタ構成で設定しているときのプライマリクラスタ
    - primarySpec
      - adminUser
        - passwordRef
          - adminUserが利用するSecret
      - allowExternalIncomingTraffic
        - 外部からのTrafficを受け入れるかどうか
      - auditLogTarget
        - audit logの連携設定
        - syslog
          - certsSecretRef
            - Syslog serverとのTLSで利用されるSecret
            - name
            - namespace
          - host 
            - syslog serverのFQDNかIPアドレス
      - availabilityOptions
        - HA構成にするかどうかやその設定
          - LlivenessProve
            - Enabled、Disabled、OpDisabledのいずれか
      - databasePatchingTimeout
        - データベースへのパッチ適用のタイムアウト時間。OracleのSTSやOPatch/datapatchとは別にカウントされる        
      - dbLoadBalancerOptions
        - "DBNetworkServiceOptions allows to override some details of kubernetes Service created to expose a connection to database." DBNetworkServiceOptionsとは？
        - DBが利用するLoad Balancerで利用するの設定。いまはgcpのみらしい
        - gcp
          - loadBalancerIP
            - LBのstatic IP
          - loadBalancerType
            - ""、Internal、Externalのいずれか
      - features
        - VertexAIを利用する場合の設定
          - googleMLExtension
            - vertexAIKeyRef
            - vertexAIKeyRegion
      - images
        - サービスエージェントなどのデータプレーンが利用するGCR image
      - isStopped
        - Instanceが停止しているかどうか
      - mode
        - InstanceがOperatorによってどう管理されるか
        - ManuallySetUpStanby、Pause、Recoveryのいずれか
      - parameters
        - DBフラグのマップ
      - replication
        - レプリケーションで利用する他DBとのコネクション設定
        - profile
          - certificateReference
            - TLSで利用されるsecretを検索するために利用するキー？
          - secretRef
            - TLSで利用されるCertificateを含むSecret
            - name
            - namespace
          - host
            - 他DBのホスト名
          - isActive
            - コネクションが利用可能かどうか
          - isSynchronous
            - コネクションが同期レプリケーションかどうか
          - name
            - profile名
          - password
            - Passwordとして利用するSecret
            - name
            - namespace
          - passwordResourceVersion
            - password secretのバージョン
          - port
            - 同期先(元)が利用しているポート
          - role
            - 同期先か同期元か
            - Upstream、Downstreamのいずれか
          - type
            - レプリケーションの種類
            - PhysicalかLogicalか
          - username
            - 同期先(元)で利用するユーザー名
      - resources
        - DBインスタンスが利用するリソース
        - cpu
          - 利用するCPU量
        - disk
          - 利用するボリュームの設定
          - accessMode
            - KubernetesボリュームのaccessMode
          - annotation
            - PVCに割り当てるannotation
          - name
            - ボリュームの利用用途
            - DataDisk、LogDisk、BackupDisk、ObsDiskのいずれか
          - selector
            - バインドするボリュームを選択するselector
            - 省略
          - size
            - 利用するボリュームの容量
          - storageClass
            - ストレージのプロビジョニングに利用するCSIドライバー
          - volumeName
            - PersistentVolume名
        - memory
          - 利用するメモリ容量
      - service
        - 顧客が選択できるオプショナルのセミ・マネージドサービス
        - とは？
      - sourceCidrRangec
        - Clientの接続を許可するCIDR
      - tls
        - DBインスタンスへのコネクションに適用するTLS
        - certSecret
          - tlsで利用するca.crt、tls.key、tls.crtを含むSecret
          - name
      - version
        - DBのバージョン
    - tls
     - DBインスタンスへのコネクションに適用するTLS
     - certSecret
       - tlsで利用するca.crt、tls.key、tls.crtを含むSecret
       - name 

- failovers
  - description: Failover represents the parameters and status of a single failover
  - parameters
    - dbclusterRef
      - フェイルオーバーを実行するDBクラスタ名
- restores
  - description: Restore is the Schema for the restores API
  - parameters
    - backup
      - リストア対象となるバックアップ
    - clonedDBClusterConfig
      - clone(リストア)先となるDBClusterのスペック。指定なしの場合sourceDBClusterと同一のスペック
      - dbclusterName
    - pointInTime
      - リストア対象のPITR。指定なしの場合最新のPITR
    - sourceDBCluster
      - リストア元となるDBCluster
- softwarelibraries
  - description: 記載なし
    APIのversionがv1とv1alphaの二つあった。パラメータとしてはDBEngineのバージョンなどがあった
    CRD自体の説明がほとんどなく、何をしているものかは不明

### `alloydbomni.dbadmin.goog/v1`を利用してみる。
~~crd.yamlから読みといた内容を元にyamlファイルを記載してみます。~~
と思っていたのですが、これらのCRDは現状正常に動作していない様に見えます。
以下のようなyamlファイルでbackupplanを作成しようとしたところ、バックアップの作成に失敗したようでした。
```
apiVersion: alloydbomni.dbadmin.goog/v1
kind: BackupPlan
metadata:
  name: alloydb-omni-poc
spec:
  PITREnabled: false
  backupLocation:
    gcsOptions:
      bucket: nakadate-alloydb-omni-bucket
      key: /dump/tmp.dmp
    type: GCS
  cronSchedule: "0 0 * * *"
  dbclusterRef: alloydb-omni-poc

```
実際に発生したエラー(Status)は以下のようなものです。
どうやら内部でホスト名として指定されている`metadata.google.internal`というアドレスが存在しないため発生しているエラーなようです。
```
Status:
  Critical Incidents:
    Code:         DBSE0000
    Create Time:  2023-12-04T16:49:35Z
    Message:      Internal. Unknown Critical Incident.
    Resource:
      Component:  controller-default
      Location:
    Stack Trace:
      Component:  controller-default
      Message:    DBSE0000: Internal. Unknown Critical Incident. rpc error: code = Unknown desc = dbdaemon/EnableBackup: add new backup repo &{0xc0010041e0}: failed to create pgbackrest stanza: failed to run pgbackrest: exit status 49, parameters [stanza-create --config-path=/backup --stanza=db], error message ERROR: [049]: unable to get address for 'metadata.google.internal': [-2] Name or service not known
           [HostConnectError] on 12 retries from 121-60010ms: unable to get address for 'metadata.google.internal': [-2] Name or service not known
```
これはユーザー側で設定出来るパラメータではないため、現状は利用出来ない機能と言って良いでしょう。
またこのほかのリソースについても同様に何かしらのエラーが発生し、リソースの作成までは出来ても実行に失敗しました。

## 感想
公式でもうすこしドキュメント整備してほしい：；
~~How to useみたいなのはAPIリファレンス見てなんとかするからCRD/OperatorのAPIリファレンスは自動生成して公開しておいて欲しい：；~~
と思っていたのですが、Oracleへの言及などよくわからないパラメータがとても大かったので利用方法のドキュメントも必須でした。
Preview版なため仕方は無いもののomniが他クラウドでも使えることをアピールしていた一方で、GCP(そういえばGCPってワードは使ってほしく無かったのでは？ )でなければ使えない機能が多いのは課題だと思いました。
またDBCluster以外の実際にデータベースを運用するのに必要となる、バックアップのスケジューリングやフェイルオーバーが失敗したりとカスタムリソースとして生成まではできても内部でエラーが発生してしまうため本格的な利用は難しいでしょう。
Oracleへの言及がなにかベースとなったOperatorに由来するものなのか？ それとも将来的に何かしらの形でOracleをサポートする予定なのかは非常に気になりました。
