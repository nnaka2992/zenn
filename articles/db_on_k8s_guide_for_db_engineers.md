---
title: "データベースエンジニアのためのDB on Kubernetes入門ガイド"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes","PostgreSQL","DB"]
published: true
published_at: 2023-12-01 08:30
---
このエントリーは[3-shake Advent Calendar 2023](https://qiita.com/advent-calendar/2023/3-shake) 1日目の記事です。

[株式会社スリーシェイク](https://3-shake.com/)のメンバーが各々自由に技術・非技術ネタを投稿するカレンダーとなります。

## はじめに
1959年にW. C. McGeeがデータベースという概念を提唱してから約65年、様々なアーキテクチャのデータベースが提案され様々なプラットフォームで利用されてきました。
古くはメインフレームを中心に動作していたデータベースは、マイコンブームとともにそのアーキテクチャを変えながらにオープン系システムへと主戦場を移して行きました。
オープン系が主流になってからもその進化は止まることなく、ベアメタルからVMに、VMからクラウドへと少しずつ形を変えてきました。
全ての組織がデータを扱う以上、データベースマネジメントシステムが無くなることはなくハードウェアや仮想化技術に追従して進化して行きます。
データベースはコンテナ技術の興隆にも対応し、コンテナオーケストレーションツールであるKubenertes上に構築することは特別なことではなくなりました。

## 対象読者
この記事は今までKubernetesに軽く触れたことはあるがKubernetes上でデータベースを動かしたことの無いデータベース技術者をメインに、Kubernetes利用経験がありクラスタ化されたデータベースがKubernetes上でどのように実現されているかの裏側に興味がある人を対象とします。
一方でここで書かれている内容はあくまでチュートリアルレベルのため、プロダクションで利用には適していません。
ここで紹介する方法はDBMSの運用に求められる多くの機能が欠如しているため、プロダクションでの利用はOperatorの使用を検討してください。

## Kubernetesとは
Kubernetes(k8s)とは複数のホストに渡ってコンテナ化されたアプリケーションを管理するオープンソースのシステムです。
Kubernetesの始まりはGoogleが社内で利用していたBorgという、何千ものジョブとアプリケーションを管理する
現在ではCloud Native Computing Foundationにより管理され、数千のコントリビューターに支えられ、数万もの開発者に利用されています。

## PostgreSQL ClusterをKubernetesに作成する

### 事前準備
今回利用するツールは[rtx](https://github.com/jdx/rtx)(asdf)というツールを利用してインストイールします。
このツールを利用することでconfigファイルに記載したバージョンのツールをインストールすることができ、環境ごとに異なるバージョンを利用することが出来ます。
以下のコマンドを実行して、rtxをインストールします。
コマンドはx64アーキテクチャのCPUで動作するLinuxを前提としているため、他環境の場合はドキュメントを参照してください。
``` bash
> curl https://rtx.pub/rtx-latest-linux-x64 > ~/bin/rtx
> chmod +x ~/bin/rtx
```
つづけてkindやkubectlなど今回利用するツールをrtxから利用出来るように登録します。
```
> rtx plugin add kind https://github.com/reegnz/asdf-kind.git
> rtx plugin add kubectl https://github.com/asdf-community/asdf-kubectl.git
```
次に以下のyamlを作業ディレクトリに`.tool-versions`というファイル名で保存します。
```yaml
kind 0.20.0
kubectl 1.28.3
```
rtxコマンドを実行して`.tool-versions`に定義したツールをダウンロードします。
```bash
> rtx install
```
最後にrtxをアクティベートします。
```bash
> rtx activate
> rtx shell
```
:::message
ツールの紹介

・[kind](https://kind.sigs.k8s.io/)

  ローカルにKubernetse Clusterを作成するツール

・[kubectl](https://kubernetes.io/ja/docs/reference/kubectl/)

  Kubernetesを操作するツール
:::
### Kubernetes環境の作成
今回はDockerコンテナをKubernetesのノードとしてKubernetes Clusterを立ち上げる[kind](https://kind.sigs.k8s.io/)というツールを利用します。
kindはGKEやEKSなどのマネージドKubernetesと違い、ローカル環境で完結するため無料で利用でき、またやminikubeでは実現出来ないマルチノードクラスターを立ち上げることが出来るため個人での検証に向いています。
まず始めに以下の内容を`kind.yaml`というファイル名で保存します。
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: postgres-cluster
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```
作成した`kind.yaml`を使用してKubernetes Clusterを作成します。
```bash
> kind create cluster --config ./kind.yaml
Creating cluster "postgres-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-postgres-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-postgres-cluster

Thanks for using kind! 😊
```
最後にkindで作成したKubernetes Clusterが正常に利用できることをkubectlコマンドを利用して確認します。
```bash
> kubectl get node
NAME                             STATUS   ROLES           AGE    VERSION
postgres-cluster-control-plane   Ready    control-plane   120m   v1.27.3
postgres-cluster-worker          Ready    <none>          120m   v1.27.3
postgres-cluster-worker2         Ready    <none>          120m   v1.27.3
postgres-cluster-worker3         Ready    <none>          120m   v1.27.3
```
ここまで異常なく実行できていれば、ローカル環境でKubernetes Clusterを利用することが出来るはずです。

### Namespaceの作成
まず始めにKubernetseにPostgreSQL Clusterを配置するNamespaceを作成します。
Namespaceとは同一の物理クラスタ上で動作する仮想クラスターです。
kindでは以下の5つが初期から作成されるNamespaceです。
```
> kubectl get namespace
NAME                 STATUS   AGE
default              Active   15h
kube-node-lease      Active   15h
kube-public          Active   15h
kube-system          Active   15h
local-path-storage   Active   15h
```
このうち`kube`で始まるのはKubernetesが内部で利用するリソースを配置するNamespaceで、`default`はKubernetesユーザーが何も指定しなかった場合に利用されるデフォルトのNamespaceです。
`local-path-storage`は他の4つと異なりkindが内部で利用するリソースを配置しています。
KubernetesではほとんどのリソースはNamespaceに属しますが、NodeやPersistentVolume(Storage)のような低レベルリソースはその限りではありません。
Namespaceを利用することでDBMSにおけるSCHEMAやDATABASEの様にリソースを論理的に分割したり、Namespaceごとにリソースクォータを設定することが出来ます。
今回はPostgreSQL関連のリソースをまとめて配置する`postgres` Namespaceを作成します。
単純なNamespaceの作成は簡単で以下の内容を`manifests/ns.yaml`というファイル名で保存します。
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
```
このファイルを`kubectl`コマンドで`apply`するだけでNamespaceを作成することが出来ます。
```bash
> kubectl apply -f manifests/ns-postgres.yaml
namespace/postgres created
```
これは必須ではありませんが、KubernetesでNamespaceに属するリソースを表示するときに初期状態だと`default` Namespaceが表示されるため毎回`kubectl --namespace postgres get pods`というように`postgres` Namespaceを明示的に選択してあげる必要があります。
`--namespace postgres`を毎回指定するのも良いですが面倒なので`postgres` Namespaceをデフォルトで利用するために以下のコマンドを実行します。
```bash
> kubectl config set-context --namespace=postgres --current
Context "kind-postgres-cluster" modified.
```
こうすることで叩いたコマンドは自動的に`--namespace postgres`を指定したと解釈してくれるようになりました。

### PostgreSQLで利用するConfigMapを作成する
`postgresql.conf`や`pg_hba.conf`などの設定ファイルとPostgreSQLの初期化で利用する[ConfigMap](https://kubernetes.io/ja/docs/concepts/configuration/configmap/)を設定します。
ConfigMapは名前から分かるとおり、設定値をKey-Valueのペアとしてあつかうリソースで設定値にはファイルに似た値や単純な文字列・数値のような値の二つがあります。
一般的にコンテナはdev/prdのように異なる環境の差異を吸収したり、後述するクラスタ化したStatefulSetでのPrimary Secondary間で異なる設定を実現する場合などに役立ちます。
今回作成するコンフィグマップは`postgresql.conf`や`pg_hba.conf`などのPostgreSQLの設定ファイルとして利用されるものと、`PGDATA`や`POSTGRES_USER`などPostgreSQLの公式Docker imageで利用される環境変数を纏めたConfigMapを用意します。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres
  labels:
    app: postgres
    app.kubernetes.io/name: postgres
    type: file
  namespace: postgres
data:
  postgresql.conf: |
    listen_addresses = '*'
    max_wal_senders = 5
    max_wal_size = 10GB
    wal_level = replica
    synchronous_commit = on
    archive_mode = on
    archive_command = 'test ! -f /archives/%f && cp %p /archives/%f'
    wal_sender_timeout = 1s
    synchronous_standby_names = 'FIRST 2(*)'
    restore_command = 'cp /archives/%f %p'
    recovery_target_timeline = 'latest'
  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    # "local" is for Unix domain socket connections only
    local   all             all                                     trust
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            trust
    # IPv6 local connections:
    host    all             all             ::1/128                 trust
    # Allow replication connections from localhost, by a user with the
    # replication privilege.
    local   replication     all                                     trust
    host    replication     all             127.0.0.1/32            trust
    host    replication     all             ::1/128                 trust
    #
    host    all             all             10.0.0.0/8              trust
    host    replication     all             10.0.0.0/8              trust
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pgenv
  labels:
    app: postgres
    app.kubernetes.io/name: postgres
    type: env
  namespace: postgres
data:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
  POSTGRES_DB: postgres
  PGDATA: /var/lib/postgresql/data/
  TZ: Asia/Tokyo
```
このように設定ファイルをConfigMapとして分離することで、設定ファイルを変更するためにコンテナイメージを更新する必要が無くなります。

### PostgreSQLのエンドポイントを構成するServiceを作成する
KubernetesでPodの集合やそれにアクセスするためのポリシーを定義するときは[Service](https://kubernetes.io/ja/docs/concepts/services-networking/service/)を利用します。

データベースではWriterとReaderで接続先のエンドポイントが分かれているようなものを実現する時に利用することが出来ます。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
    app.kubernetes.io/name: postgres
spec:
  ports:
  - name: postgres
    port: 5432
  clusterIP: None
  selector:
    app: postgres
```

```
> kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
postgres        ClusterIP   None           <none>        5432/TCP   3h48m
```
上のServiceはHeadless Serviceと呼ばれるものでIPアドレスを持ちません。
selectorにマッチするpodに対してのみリクエストをルーティングするDNS名を提供します。

### PostgreSQL Clusterを構成するStatefulSetを作成する
PostgreSQLのPrimary Secondary構成をStatefulSetを利用して作成します。
データベースクラスタの作成に利用していることからも分かるとおり、StatefulSetはデータベースやKafka、Zookeeperのような永続ブローカーをホストするのに適しています。
StatefulSetはDeploymentなどに比べ、一意なネットワーク識別子の提供や順序の安定したデプロイとスケーリング規則的なローリングアップデートなどのStatefulSetのpod間の依存関係に配慮した機能を多数提供しています。
KubernetesのpodにはinitContainerという通常のコンテナを起動する前に実行されるコンテナがあります。
StatefulSetはpod名が規則的になることを利用してPrimary Secondaryで異なる起動処理を行なわせることが出きます。
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  serviceName: postgres
  replicas: 3
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
      - name: init-permission
        image: busybox:1.36.1
        command:
        - chown
        - -R
        - "70"
        - /var/data/
        - /archives/
        volumeMounts:
        - name: pgdata
          mountPath: /var/data/
        - name: archives
          mountPath: /archives/
      - name: init-postgres
        image: postgres:15.5-alpine
        securityContext:
          runAsUser: 70
        envFrom:
        - configMapRef:
            name: pgenv
        command:
         - bash
         - "-c"
         - |
           set -ex
           # サーバがPrimaryかReplicaか判別する
           [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
           ordinal=${BASH_REMATCH[1]}
           ################################################################################
           # Primaryサーバでデータが存在しない場合データベースを初期化する
           ## 1. initdbでpostgresクラスタをセットアップする
           ## 2. postgresql.confとpg_hba.confをコピーする
           ## 3. 初期化を終了しexitする
           ################################################################################
           [[ -z $(ls -A $PGDATA) ]] && [[ ${ordinal} -eq 0 ]] &&\
           initdb -U $POSTGRES_USER --pwfile=<(printf "%s\n" "$POSTGRES_PASSWORD") &&\
           cp /mnt/config-map/postgresql.conf ${PGDATA} &&\
           cp /mnt/config-map/pg_hba.conf ${PGDATA}
           [[ ${ordinal} -eq 0 ]] && exit 0
           ################################################################################
           # Secondaryサーバにデータが存在しなければ初期化を行なう
           ## 1. Primaryからバックアップを取得
           ## 3. postgresql.confにSecondary固有設定を追加
           ################################################################################
           [[ -n $(ls -A $PGDATA) ]] && exit 0
           pg_basebackup -D /var/lib/postgresql/data/ -h postgres-0.postgres -p 5432 -U postgres -Xs -R -P
           echo "restore_command = 'cp /archives/%f %p'" >> $PGDATA/postgresql.conf
           echo "recovery_target_timeline = 'latest'" >> $PGDATA/postgresql.conf
        volumeMounts:
        - name: postgresql-volume
          mountPath: /mnt/config-map/
          readOnly: true
        - name: pgdata
          mountPath: /var/lib/postgresql/data/
      volumes:
      - name: postgresql-volume
        configMap:
          name: postgres
          items:
          - key: postgresql.conf
            path: postgresql.conf
          - key: pg_hba.conf
            path: pg_hba.conf
      containers:
      - name: postgres
        image: postgres:15.5-alpine
        securityContext:
          runAsUser: 70
        envFrom:
        - configMapRef:
            name: pgenv
        ports:
        - name: postgres
          containerPort: 5432
        volumeMounts:
        - name: postgresql-volume
          mountPath: /mnt/config-map/
          readOnly: true
        - name: pgdata
          mountPath: /var/lib/postgresql/data/
        - name: archives
          mountPath: /archives/
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["pg_isready"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["psql", "-h", "127.0.0.1", "-U", "postgres",  "-c", "SELECT 1" ]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      volumes:
      - name: postgresql-volume
        configMap:
          name: postgres
          items:
          - key: postgresql.conf
            path: postgresql.conf
          - key: pg_hba.conf
            path: pg_hba.conf
      - name: archives
        persistentVolumeClaim:
          claimName: archives
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

ここまでの設定を問題なく反映できていれば、PostgreSQLクラスターが立ちあがっているはずです。

## DBaaSを実現するPostgreSQL Operatorの紹介
マネージドデータベースやDBaaSが提供する機能は様々ですが、最低限期待される機能は以下のようなものがあります。
1. データベースクラスタのデプロイ
2. フェイルオーバー
3. バックアップとリカバリ
4. メトリクスの収集

これらの機能は運用を自動化し、データベースエンジニアがより時間をかけるべきタスクに注力できるようになります。
残念ながら今回紹介したStatefulSetを利用したデータベースクラスタでは2つめのフェイルオーバーを実現することが難しいです。
またその他の項目についても作りこめば十分実現可能ではあるいっぽうで、データベースに求められる機能や考慮すべき項目は多岐にわたるため一々リソースを定義するのは非常に大変です。
そこでDBaaSを実現するOperatorを紹介します。

### Zalando Postgres Operator
ZalandoというヨーロッパのファッションECを運営している会社が公開しているオペレーターです。
性能監視などの機能を削り、最低限のクラスタ管理機能のみを提供しています。
PostgreSQLのOperatorとしては最も古いものの1つで、2023年12月1日現在もっとも人気(GitHubのスターが多い)Operatorでもあります。

### PGO
Crunchy Dataという商用PostgreSQLソリューションを提供している会社が開発しているオペレーターです。
PostgreSQLを専門にあつかう会社が開発しているだけあり、クラスタの管理からリストア、モニタリング機能までクラウドのマネージドデータベースに負けない機能を提供します。
ZalandoのOperatorについで人気なOperatorで計測期間によってはZalandoのOperatorをしのぐときもありました。

## まとめ
技術選定は常にメリットとデメリットのトレードオフです。
データベースをKubernetesで動かす試みは発展途上な部分も多く、全てのユースケースにマッチするとは言えません。
しかし前述の通りデータベースはこれまであらゆる実行環境に適合し進化を続けてきました。
次の10年にはKubernetesでデータベースを動かすことは当りまえになっているに違いありません。
本当はまだまだpgpoolのDeploymentを作ったり、Prometheus/Grafanaでデータベースの監視環境を作ったり、CronJobで定期的なバックアップを取得したりなど書きたいことがあったのですが間に合いませんでした。
12月1日になってから書いている部分が複数あるのでOperatorまわりなどもとても記載が薄いです。
そのうち加筆します。
明日は[@nwiizo](https://twitter.com/nwiizo)による「2023年 俺が愛した本たち」です。

## 参考
- [rtx](https://github.com/jdx/rtx#readme)
- [kind](https://kind.sigs.k8s.io/)
- [StatefulSet Basics](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Run a Replicated Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/#services)
- [Kubernetes を用いた PostgreSQL の運用](https://www.sraoss.co.jp/wp-content/uploads/files/event_seminar/material/2020/dbts034_session_pg_on_k8s.pdf)
- [PostgreSQLをKubernetes上で活用するためのOperator紹介！](https://www.slideshare.net/nttdata-tech/postgresql-kubernetes-operator-cloud-native-database-meetup-3-nttdata)
