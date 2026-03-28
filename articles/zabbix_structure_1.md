---
title: "Zabbix構築ログ① ～基本的なサーバー構築とエージェントのインストール～"
emoji: "🚥"
type: "tech"
topics: ["Zabbix", "インフラ構築", "監視", "エンジニア初心者"]
published: true
---

# はじめに
Zabbixの構築を検証、本番環境へ投入したため、自分の復習のために手順を残します。
ホスト登録は別の記事にして、とりあえずZabbixのインストールとログイン確認、Agentのインストールまで行います。

# Zabbixインストール
## 1.	リポジトリのアップデート
以下のコマンドを実行し、RedHatOSを最新の状態にします。
```
sudo -i
dnf update
```

## 2.	リポジトリの登録
Zabbixのリポジトリからインストーラーをダウンロードして実行します。
```
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/rhel/9/noarch/zabbix-release-latest-7.4.el9.noarch.rpm
dnf clean all
```

## 3.	Zabbix Server、関連モジュールのインストール
Zabbixのリポジトリからインストーラーをダウンロードして実行します。
```
dnf install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent
```	

## 4.	MySQL、Apache2のインストール
MySQLのリポジトリを追加します。
```
dnf install https://dev.mysql.com/get/mysql84-community-release-el9-2.noarch.rpm
dnf update
```

MySQLとApache2をインストールし、自動起動設定と有効化を行います。
```
dnf install mysql-server httpd -y
systemctl start mysqld httpd
systemctl enable mysqld httpd
```

## 5.	MySQLのrootユーザー設定
初めにMySQLへルートユーザーでアクセスするために、ルートユーザーのパスワードを変更します。
```
mysqladmin -u root password
New password: Dbro0t!*
Confirm new password: Dbro0t!*
Warning: Since password will be sent to server in plain text, use
ssl connection  to ensure password safety.
```

権限エラーの場合は、以下のコマンドで暫定パスワードを確認します。
```
grep 'temporary password' /var/log/mysqld.log
```

## 6.	MySQLデータベース作成
MySQL上で、Zabbixにて使用するデータベースの作成と、ユーザー作成と権限付与を行います。
```
mysql -u root -p
Enter password: Dbro0t!*
```

暫定パスワードの場合は、パスワードを変更します。
```
ALTER USER root@localhost IDENTIFIED BY 'Dbro0t!*';
```

データベースを作成、権限付与していきます。
```
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
Query OK, 1 row affected (0.03 sec)
mysql> create user zabbix@localhost identified by ' Dbu5er!*';
Query OK, 0 rows affected (0.05 sec)
mysql> grant all privileges on zabbix.* to zabbix@localhost;
Query OK, 0 rows affected (0.03 sec)
mysql> set global log_bin_trust_function_creators = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> quit;
Bye
```

## 7.	MySQLの初期設定
初期スキーマ、データを入力します。（解凍せずにリストアする方法）
```
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
```

MySQLの設定を変更します。
```
mysql -uroot -p
Enter password:Dbro0t!*
mysql> set global log_bin_trust_function_creators = 0;
Query OK, 0 rows affected, 1 warning (0.01 sec)
mysql> quit;
Bye
```

## 8.	Zabbix.confの編集
Zabbixのconfigファイルをバックアップします。
```
cp -p /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf.org
```

configファイルにデータベースへアクセスできるユーザー情報を追記します。
```
vi /etc/zabbix/zabbix_server.conf
DBPassword= の#を外し、DBPassword=Dbu5er!* に変更
:wqで保存
```

変更された内容を差分で確認します。	
```
diff /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf
< DBPassword=Dbu5er!*
---
> # DBPassword=
```

## 9.	Zabbixの再起動
Zabbixのサービスを再起動し、自動起動するように設定します。
```
systemctl restart zabbix-server zabbix-agent httpd
systemctl enable zabbix-server zabbix-agent
```

# セキュリティ設定の変更
## 1.	10050.tcpと10051/tcp、HTTPポート開放
Firewalldサービスが起動している場合は、ポート開放を行います。
```
Systemctl status firewalld
# Enable なら以下設定
Firewall-cmd –-add-port=10050/tcp –permanent
Firewall-cmd –-add-port=10051/tcp –permanent
Firewall-cmd –-add-port=80/tcp –permanent
Firewall-cmd –-add-port=443/tcp –permanent
Systemctl restart firewalld
```

## 2.	SELinuxの無効化
予期せぬ動作を回避するため、SELinuxを無効化します。
```
Vi /etc/selinux/conf
Selinux = enforcing
Selinux = disabled　に変更
Systemctl daemon-reload
```

# Zabbix 7.4 へのGUIアクセス
## 1.	Zabbix GUIへアクセス
ブラウザから、以下のURLでアクセスします。
**http://<IPアドレス>/zabbix**

2.	初期設定
日本語が選択できる場合は選択し、不可の場合は英語で進めます。
![](https://storage.googleapis.com/zenn-user-upload/d923af8c2031-20260329.png) 

設定に間違いがないかチェックが実行され、すべてOKであれば次に進みます。
![](https://storage.googleapis.com/zenn-user-upload/73c598d43ece-20260329.png)

今回はMySQLを選択し、DBユーザー名も既定値で設定しているため、そのままで進めます。
Passwordの部分には、今回設定した 「Dbu5er!*」 を入力します。
![](https://storage.googleapis.com/zenn-user-upload/615c3fd3294a-20260329.png)

サーバー名を入力し、タイムゾーンは 「(UTC+09:00) Asia/Tokyo」 を選択します。
![](https://storage.googleapis.com/zenn-user-upload/e35e0b580cb6-20260329.png)

設定が完了すると、サインイン画面に進みます。
![](https://storage.googleapis.com/zenn-user-upload/405abb0b7224-20260329.png)

デフォルトユーザーを使用してサインインします。
ユーザー名：Admin
パスワード：zabbix
![](https://storage.googleapis.com/zenn-user-upload/eefc146c8086-20260329.png)

# Zabbix 7.4 の日本語化設定
## 1.	ロケールの設定確認
ZabbixのGUIから日本語が選択できない場合、Localeの設定を追加します。
```
locale -a
# ja_JP.UTF8 UTF8 がない場合は以下を実行
vi /etc/locale.gen
# 変更前: #ja_JP.UTF-8 UTF-8 
# 変更後: ja_JP.UTF-8 UTF-8
locale-gen
systemctl restart apache2
```

# Zabbix Agent2 7.4.2のインストール（Windows）
## 1.	Zabbix Agent2のダウンロード
以下のURLから、Zabbix Agent2をダウンロードします。
https://www.zabbix.com/jp/download_agents
![](https://storage.googleapis.com/zenn-user-upload/cea0a941cd09-20260329.png)

ダウンロードしたインストーラーを実行します。
![](https://storage.googleapis.com/zenn-user-upload/dce4ebfa365f-20260329.png)

利用規約に同意します。
![](https://storage.googleapis.com/zenn-user-upload/491fd3dd732b-20260329.png)

カスタムセットアップ画面では変更せず進めます。
![](https://storage.googleapis.com/zenn-user-upload/70ef4842c4dd-20260329.png)

「Hostname」 には、端末のホスト名を入力し、今回は「AD2025-1」 とします。
「Zabbix Server IP/DNS」には、接続を許可するZabbixサーバーの情報を入力します。
「Server or Proxy for active checks」には、情報の送信先Zabbixサーバーの情報を入力します。
暗号化の設定を行う場合は、「Enable PSK」 を有効化します。
「Add Agent Location to the PATH」 にチェックを入れると、インストールと同時にZabbixエージェントの場所をシステムPATH変数に追加することができます。
![](https://storage.googleapis.com/zenn-user-upload/75c2711c00f7-20260329.png)

「Install」 をクリックすると、インストールが開始されます。
![](https://storage.googleapis.com/zenn-user-upload/7d4dc1429d83-20260329.png)

インストールが完了すると、以下の画面に遷移します。
![](https://storage.googleapis.com/zenn-user-upload/7bcd2c1c78e6-20260329.png)

# Zabbix Agentのインストール（Linux）
## 1.	リポジトリの追加
はじめに、Zabbix Packageのリポジトリを追加します。
```
rpm -Uvh https://repo.zabbix.com/zabbix/<Zabbixバージョン>/release/<OS>/<OSバージョン>/noarch/zabbix-release-<Zabbixマイナーバージョン指定>-<Zabbixバージョン>.<OSバージョン>.noarch.rpm
# （例）rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/alma/10/noarch/zabbix-release-latest-7.4.el10.noarch.rpm →Zabbix7.4の最新、Redhat10の場合
```

一度リポジトリのキャッシュを削除します。
```
dnf clean all
```

## 2.	Zabbix-Agentのインストール
Packageの中から、Zabbix-Agentのみをインストールします。
```
dnf install zabbix-agent -y
```

インストールが完了したら、Zabbix-agentd.confを編集し、サーバーと接続できるように設定します。
```
Cp zabbix-agentd.conf zabbix-agentd.conf.org
Vi zabbix-agentd.conf

# Zabbix Agentの場合
#### Passive checks related
Server=<ZabbixサーバーのIPアドレス>

# Zabbix Agent 2の場合
#### Active checks related
ServerActive=<ZabbixサーバーのIPアドレス>
Hostname＝<Zabbixサーバーのホスト名>
```

インストールが完了したら、サービスの起動と自動起動設定を入れておきます。
```
Systemctl start zabbix-agent
Systemctl enable zabbix-agent
```

10050/tcp、10051/tcpを開放しておきます。
```
Firewall-cmd –-add-port=10050/tcp -–permanent
Firewall-cmd –-add-port=10051/tcp –permanent
Firewall-cmd –-reload
Firewall-cmd –-list-all
```

その後、監視ホストにZabbixAgentもしくはZabbixAgentActiveのテンプレートを適用すると、Zabbixサーバーから疎通を確認することができます。（ZBXが緑色表示）
※ZabbixPackageインストール：https://www.zabbix.com/download

# Zabbix AgentとZabbix Agent 2の違い
ZabbixにはAgentとAgent2が現在あり、いくつかの違いがあります。
今回はAgentをインストールしましたが、Agent2でも動作は特に変化ありませんでした。

また、以下のような違いがあります。
* **Zabbix Agent**：C言語、多数のOSに対応、アクティブチェックが順次実行、デーモン化が可能
* **Zabbix Agent2**：Go言語、LinuxとWindowsのみ、Dockerなども監視対象に入れられる、アクティブチェックが並列実行可能

* Agent 2のメリット
    * より多くの監視対象にネイティブ対応（Docker, DBなど）
    * プラグインによる柔軟な拡張性
    * 同時実行性が高く、負荷分散に優れる
* Agentのメリット
    * 軽量で安定性が高い
    * 古いOSや特殊な環境でも動作可能
    * C言語によるモジュール開発が可能

# Zabbix Agent パッシブ/アクティブの違い
また、Zabbix Agentにはパッシブ、アクティブの2つのモードが存在します。
今回の本番環境では、定期的な死活監視のみとなったため、処理の軽いパッシブモードでのインストールを行いました。
その際には、Agent側のポート開放も忘れずに行う必要があります。

* **パッシブモード**：Zabbixサーバーがエージェントに問い合わせて情報を取得、エージェント側の 10050 ポートを開放する必要あり
* **アクティブモード**：Zabbix Agentが自分からサーバーに接続して監視データを送信、NAT環境・インターネット越しの監視に有利

# おわりに
一度、転職前に自宅で構築したことはありましたが、UIの表示とホスト登録で満足していました。

実際に構築に入ってみて、Agentの種類やインストール手順の再現性を出すことができ、少し成長したかなと思います。
一方で、この先出てくる有効アイテムの制限やマップの作製は、実際に構築に入らないと一生やることがなかったのではないかと感じていて、
ここまでの手順を理解できるようになれて、少しはエンジニアに近づけたかなと。。。

実際に動くだけではダメ、インストールが検証通りスムーズにできたうえで、その先が重要なのだと思い知りました。
次はホストの登録をまとめておきたいと思います！