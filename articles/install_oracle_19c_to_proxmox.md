---
title: "Oracle Dataabse 19cの検証環境が欲しいからProxmoxに環境構築する"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["db", "Oracle"]
published: true
---

# 概要

300年ぶりぐらいに、ローカル環境(非Cloud環境)でホストしたOracle Databaseが欲くなったので、自宅にあるProxmoxへインストールします。

# 前提

- Proxmoxにダウンロード済みのOracle Linux 9のイメージを利用する。
- 利用するOracle Databaseは19cとする。
- **検証環境のため本番用途に適した設定ではない。**

# Proxmox VMを建ち上げる
## Oracle Database 19cのサーバ要件
今回関係あるもののみ抜粋しています。
- OS
  - Oracle Linux 9およびRed Hat互換カーネル: `5.14.0-70.22.1.0.2.el9_0.x86_64`以上
- RAM
  - 1GB以上
- Storage
  - 7.2GB以上

## ProxmoxのVMの構成
上記で確認したサーバ要件に抵触しないよう配慮したうえで、Oracle DatabaseをホストするVMを作成します。^[Proxmox nodeは余裕があるので最低要件にはしません。]

- CPUアーキテクチャ
  - x86_64
- CPU core数
  - 4 vCPU
- RAM
  - 8GB
- Storage
  - 128GB of SSD

![Proxmoxの設定](/images/install_oracle_19c_to_proxmox/proxmox/config.png)

## Oracle Linuxの設定
Oracle Linuxのインストーラに従って、サーバの構成を進めます。

### 言語の設定

日本語でも良いのですが、Oracle Databaseのインストール時や利用時に文字化けが発生すると面倒なので英語で利用します。
![Oracle Linuxの言語設定](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/1_language.png)

### `LOCALIZATION > Time & Date`の設定

言語を設定するとOracle Linuxの設定画面にとびます。
![Oracle Linuxの設定画面](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/2_config_main.png)

言語を英語(United States)にしているためタイムゾーンがニューヨークとなっています。
日本在住なのでタイムゾーンを東京に変更します。

画面右上の`Region`と`City`のみ変更します。
![Time Zoneの設定画面](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/3_time_and_zone.png)

### `USER SETTINGS > Root Password`の設定

次にRootアカウントのパスワードを設定します。
任意のパスワードを設定し、`Lock root account`のみチェックを入れます。^[sshでアクセスすることはないので設定しません。]
![Root Passwordの設定画面](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/4_root_password.png)

### `USER SETTINGS > Create User`の設定

次にRootアカウントのパスワードを設定します。
Advancedの設定を除いて、チェックボックスを含めすべて入力します。
![ユーザー作成の設定画面](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/5_create_user.png)

### `SOFTWARE > Software Selection`の設定

初期インストールするソフトウェアを設定します。

今回は`Base Environment`を`Server`にし、追加のソフトウェアは指定しません。
![ソフトウェアの設定画面](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/6_software_selection.png)

### `SYSTEM > Install Destination`の設定

インストール先のディスクの設定がありますが、特に設定はしません。
`Automatic Partitioning`がデフォルトで設定されているので、そのまま利用します。
![インストール先の設定画面](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/7_install_destination.png)

### インストールの実行

ここまで設定が完了したら`Begin Installation`からOracle Linuxのインストールを開始します。
しばらくするとインストールが完了するので、システムをリブートします。
![インストールの完了画面](/images/install_oracle_19c_to_proxmox/setup_oracle_linux/7_install_destination.png)
    
# Oracle Database 19cをインストールする
## ProxmoxにOracle Databaseのrpmファイルを配布する

Proxmox VMに接続できる環境で以下を実行します。

### Oracle Database rpmファイルのダウンロード

Oracle Databaseの[ダウンロードページ](https://www.oracle.com/database/technologies/oracle-database-software-downloads.html#db_free)にアクセスし、rpmファイルをダウンロードします。
 
### Proxmoxマシンにrpmファイルを配置

ダウンロードしたrpmファイルをscpコマンドで転送します。
```shell
scp ${RPM_FILE_PATH} ${ORACLE_LINUX_USER_NAME}@${ORACLE_LINUX_IP_ADDRESS}:~/
```

### 作成したProxmox VMにOracle Databaseをインストール


なにはともあれ、インスタンスにログインしておきます。^[普通にLinuxにログインするのと同じなので省略します。]

#### rootユーザーへの切り変え

一般ユーザーでログインしているので、rootユーザーにスイッチします。

```shell
sudo su -
```

#### `Preinstall`の実行

続けてプレインストールを実行します。
このコマンドを実行すると依存するソフトウェアを一括でダウンロードしてくれるのでうれしいです。
```shell
dnf install -y oracle-database-preinstall-19c
``` 

#### Oracle Databaseのインストール
   
インストールフェーズの最後として、Oracle Databaseのインストールを実行します。
```shell
mv /home/${ORACLE_LINUX_USER_NAME}/${RPM_FILE_NAME} /tmp
cd /tmp
dnf localinstall ${RPM_FILE_NAME}
```

## Oracle Databaseの起動
###  デフォルト設定でデータベースを構築

以下のコマンドを実行し、Oracle Databaseを構築する。
```shell
/etc/init.d/oracledb_ORCLCDB-19c configure
```

### `oracle`ユーザー用の環境設定を追加

`oracle`ユーザーにスイッチし、環境変数を設定したのち、接続確認を行います。
```shell
su - oracle
cat <<EOF >> ~/.bash_profile
export ORACLE_SID=ORCLCDB
export ORACLE_BASE=/opt/oracle/oradata
export ORACLE_HOME=/opt/oracle/product/19c/dbhome_1
export PATH=$PATH:$ORACLE_HOME/bin
EOF

sqlplus / as sysdba
```
<!-- textlint-disable preset-ja-technical-writing -->
:::message
SQL*Plus: Release 19.0.0.0.0 - Production on Sun May 19 19:12:10 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL>
:::
<!-- textlint-enable preset-ja-technical-writing -->

### 自動起動用の設定を追加

#### `/etc/oratab`の設定変更

以下のコマンドを実行し、`/etc/oratab`の設定を変更します。
ここではnanoを利用していますが、viなど任意のエディタで良いです。
```shell
nano /etc/oratab 
```
内容は以下のとおりです。
最終行を以下にします。
```text
ORCLCDB:/opt/oracle/product/19c/dbhome_1:Y
```

#### 環境変数を設定する

サービスが利用する環境変数を設定します。
```shell
cat <<EOF > /etc/sysconfig/ORCLCDB.conf
ORACLE_BASE=/opt/oracle/oradata
ORACLE_HOME=/opt/oracle/product/19c/dbhome_1
ORACLE_SID=ORCLCDB
EOF
```

#### リスナサービスの設定をする

Oracle Databaseへの接続を管理するリスナをサービスとして登録に必要な設定します。
```shell
cat <<EOF > /usr/lib/systemd/system/ORCLCDB@lsnrctl.service
[Unit]
Description=Oracle Net Listener
After=network.target

[Service]
Type=forking
EnvironmentFile=/etc/sysconfig/ORCLCDB.conf
ExecStart=/opt/oracle/product/19c/dbhome_1/bin/lsnrctl start
ExecStop=/opt/oracle/product/19c/dbhome_1/bin/lsnrctl stop
User=oracle

[Install]
WantedBy=multi-user.target
EOF
```

#### データベースサービスの設定をする

Oracle Databaseをサービスとして登録に必要な設定をします。
```shell
cat <<EOF > /usr/lib/systemd/system/ORCLCDB@oracledb.service
[Unit]
Description=Oracle Database service
After=network.target lsnrctl.service

[Service]
Type=forking
EnvironmentFile=/etc/sysconfig/ORCLCDB.conf
ExecStart=/opt/oracle/product/19c/dbhome_1/bin/dbstart $ORACLE_HOME
ExecStop=/opt/oracle/product/19c/dbhome_1/bin/dbshut $ORACLE_HOME
User=oracle

[Install]
WantedBy=multi-user.target
EOF
```

#### サービスを有効化する

ここまで記載した設定をsystemctlで有効化します。
```shell
systemctl daemon-reload
systemctl enable ORCLCDB@lsnrctl ORCLCDB@oracledb
```

#### OSを再起動する

ProxmoxのコンソールからVMを再起動します。
    
#### サービスの状態を確認する
   
`systemctl statue`コマンドを実行し、サービスの状態を確認します。
```shell
systmctl status ORCLCDB@lsnrctl
```

:::message
● ORCLCDB@lsnrctl.service - Oracle Net Listener
     Loaded: loaded (/usr/lib/systemd/system/ORCLCDB@lsnrctl.service; enabled; preset: disabled)
     Active: active (running) since Sun 2024-05-19 22:20:40 JST; 3min 29s ago
    Process: 779 ExecStart=/opt/oracle/product/19c/dbhome_1/bin/lsnrctl start (code=exited, status=0/SUCCESS)
   Main PID: 1596 (tnslsnr)
      Tasks: 2 (limit: 47006)
     Memory: 66.8M
        CPU: 48ms
     CGroup: /system.slice/system-ORCLCDB.slice/ORCLCDB@lsnrctl.service
             └─1596 /opt/oracle/product/19c/dbhome_1/bin/tnslsnr LISTENER -inherit
...(中略)
May 19 22:20:40 localhost.localdomain systemd[1]: Started Oracle Net Listener.
:::

```shell
systemctl status ORCLCDB@oracledbsystemctl status ORCLCDB@oracledb
```

:::message
● ORCLCDB@oracledb.service - Oracle Database service
     Loaded: loaded (/usr/lib/systemd/system/ORCLCDB@oracledb.service; enabled; preset: disabled)
     Active: active (running) since Sun 2024-05-19 22:20:54 JST; 8min ago
    Process: 780 ExecStart=/opt/oracle/product/19c/dbhome_1/bin/dbstart /opt/oracle/product/19c/dbhome_1 (code=exite>      Tasks: 77 (limit: 47006)
     Memory: 3.1G
        CPU: 19.333s
     CGroup: /system.slice/system-ORCLCDB.slice/ORCLCDB@oracledb.service
             ├─1727 ora_pmon_ORCLCDB
             ├─1729 ora_clmn_ORCLCDB
...(中略)
May 19 22:20:40 localhost.localdomain dbstart[1634]: Processing Database instance "ORCLCDB": log file /opt/oracle/pr>
May 19 22:20:54 localhost.localdomain systemd[1]: Started Oracle Database service.
:::

#### `sqlplus`から動作を確認する
   
最後にユーザーをoracleへ切り替えて、`sqlplus`経由で動作していることを確認します。
```shell
su - oracle
sqlplus / as sysdba
```

:::message
SQL*Plus: Release 19.0.0.0.0 - Production on Sun May 19 22:33:33 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> 
:::

# まとめ

昔CentOSでOracleをダウンロードしたときはOSパラーメータを調整したり、依存するソフトを整理したり面倒だったのですが、`dnf`でインストールすると簡単なんですね。
Oracle Linuxのインストール設定をするほうが面倒でした。

# 参考
- [x86-64用のサポート対象Oracle Linux 9ディストリビューション](https://docs.oracle.com/cd/F19136_01/ladbi/supported-oracle-linux-9-distributions-for-x86-64.html#GUID-C8D5D07F-6FE1-461F-8218-5C188132BD80)
- [Oracle Databaseインストールのサーバー・ハードウェアのチェックリスト](https://docs.oracle.com/cd/F19136_01/ladbi/server-hardware-checklist-for-oracle-database-installation.html#GUID-D311E770-9444-45D0-A122-6491D1B66B8A)
- [Oracle Linux yum Serverを使用したOracle Database Preinstallation RPMのインストール](https://docs.oracle.com/cd/F19136_01/ladbi/installing-oracle-linux-with-public-yum-repository-support.html#GUID-190BAEE2-2B77-4AA2-AA6B-5D6AF73A4005)
- [10.6. systemd のユニットファイルの作成および変更 Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/system_administrators_guide/sect-managing_services_with_systemd-unit_files)

