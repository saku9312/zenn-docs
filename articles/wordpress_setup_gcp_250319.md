---
title: "WordPressをGoogle CloudのEC2で構築する"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
もともとDocker上のコンテナでWordPressを立ち上げていたが、SSL化のところで手間取ったので、まずはコンテナを使わずにローカルですべて一から入れてみた。

## 構築環境
- Google Cloud EC2
- Ubuntu 24.04LTS
- nginx 1.24.0
- mariadb 10.11.8
- php

## EC2のFree Tier（無料枠）
Google Cloudでは常時無料枠があり、以下の条件に当てはまれば基本的に費用はかからない。
- CPUがe2-micro
- リージョンがus-west1（オレゴン）,us-central1（アイオワ）,us-east1（サウスカロライナ）のどれか
- 標準永続ディスクで合計30GB以内（コンテナ複数合計でも可）
- ネットワーキングのOutが1GB以内

あとは固定のグローバルIPも取得可能だが、動いているコンテナに割り当てていない（動作していない）と費用がかかることに注意。

## EC2のVMインスタンス作成
今回は以下の構成でまず作成してみる。
- リージョン：us-west1
- CPU：e2-micro
- ディスク：15GB
- イメージ：Ubuntu 24.04LTS
- 固定IP

先にIPアドレスを予約しておく。
- VPCネットワーク＞IPアドレス＞外部静的IPアドレスを予約します　から設定。
名前は適当で、リージョンはインスタンスと合わせる。（どっちでもいいけれど）
![](https://storage.googleapis.com/zenn-user-upload/36032548f9cc-20250320.png)

↓↓↓ここからインスタンスの設定

1. Google Cloud コンソールから、VMインスタンス＞インスタンスを作成
![](https://storage.googleapis.com/zenn-user-upload/a7c94497b23c-20250320.png)

2. リージョンとゾーンの指定
ゾーンは特にどれでもいいみたい。
![](https://storage.googleapis.com/zenn-user-upload/f5f371a4d677-20250320.png)

3. CPUの選択
E2を選択してから、プルダウン内の「e2-micro」を選択する。
![](https://storage.googleapis.com/zenn-user-upload/25d79b4c74f5-20250320.png)

4. OSとストレージの指定
ここでタイプを「標準永続ディスク」にすることを忘れずに！
間違えると課金されます。
![](https://storage.googleapis.com/zenn-user-upload/109dd2959101-20250320.png)

5. ファイアウォールの指定
WordPressは最初HTTPでのアクセスになるため、HTTPのトラフィックも許可しておく。
![](https://storage.googleapis.com/zenn-user-upload/5f408163527d-20250320.png)

6. 固定IPの設定
ネットワーキングの中の下部にインターフェイスの設定ができるので、外部IPの欄から作成したIPを割り当てる。
![](https://storage.googleapis.com/zenn-user-upload/bb0e9b2ec100-20250320.png)

これで待っているとインスタンスが自動で作成される。
この後はSSHで接続して、まず全体的なアップデートをかけておく。
![](https://storage.googleapis.com/zenn-user-upload/1a7edde38089-20250320.png)

## スワップ領域の作成
EC2で立ち上げたインスタンスでは、スワップ領域が作成されていない。
メモリも1GBで心もとないので、プラスで1GBスワップ領域を作成しておきたい。

1. スワップ領域の状態を確認
```
free -h
```
2. ディレクトリの割り当てとスワップサイズの指定
bs=1M と count=1000 の組み合わせで1GBとする。
```
dd if=/dev/zero of=/swapfile bs=1M count=1000 status=progress
```
3. スワップファイルのパーミッションを変更
これをやっておかないと正常に動作しないことがあった。
```
chmod 600 /swapfile
```
4. スワップ領域としての指定と有効化
```
mkswap /swapfile
swapon /swapfile
```
5. /etc/fstabにスワップ領域を記載し、自動マウントするように設定
```
vi /etc/fstab
    /swapfile none swap sw 0 0
```
6. マウントされているか確認
```
free -h
```

## nginxのインストール
Webサーバーとして、今回はnginxを使用する。
インストールするバージョンは特に指定しないことにした。

1. nginxのインストール
```
sudo apt-get install nginx -y
```
2. nginxの起動確認
```
sudo systemctl status nginx
```
3. nginxでのページが作成されているか確認
たまにブラウザまでの表示でエラーに落ちて確認できないことがあるので、
コマンドでローカル内で確認することで、最低限ページができているかをチェック。
```
curl localhost
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
```
4. バージョン確認
やらなくてもいいけど、備忘録。
```
sudo nginx -v
    nginx version: nginx/1.24.0 (Ubuntu)
```

## mariadbのインストールと初期設定
今回はmariadbで構築してみる。
データベースの知識が薄いので、いろんなデータベースを使ってみたい。

1. mariadbのインストール
```
sudo apt-get install mariadb-server mariadb-client -y
```
2. mariadbの確認と起動
```
sudo systemctl status mariadb
sudo systemctl start mariadb```
```
3. mariadbのコンソールに入る
ここでバージョンも確認できるので、一緒に見ておく。
```
sudo mariadb -v
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 31
    Server version: 10.11.8-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04
MariaDB [(none)]>
```
4. mariadb上に新規テーブルを作成する
```
MariaDB [(none)]> CREATE DATABASE wp;
Query OK, 1 row affected (0.044 sec)
```
5. mariadb上にユーザーを作成する
ユーザーの指定は、'ユーザー名'@'locaohost' IDENTIFIED BY 'パスワード'で行う。
このシングルクォーテーションは必須。
```
MariaDB [(none)]> CREATE USER 'ユーザー名'@'localhost' IDENTIFIED BY 'パスワード';
Query OK, 0 rows affected (0.072 sec)
```
6. 新規テーブルに先ほど作成したユーザーの権限を追加する
```
MariaDB [(none)]> GRANT ALL ON wp.* TO 'ユーザー名'@'localhost' WITH GRANT OPTION;
Query OK, 0 rows affected (0.004 sec)
```
7. 権限情報を更新
```
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.003 sec)
```

ここまでで、mariadb上でWordPress用のデータベースが作成できた。
このデータベース名、ユーザー名、ホスト名（localhost）、パスワードはWordPressの初期設定時に必要になるので控えておく。

## PHPのインストール
phpが必要になるのでインストールを行う。
NginxでPHPを使用するにはPHPサポートが必要になるので、必要なPHPモジュールをインストールする。

1. 最新のphpをインストールするためにリポジトリを追加する
```
sudo apt-add-repository ppa:ondrej/php
```
2. phpをリポジトリ経由でインストール
PHPはバージョン7以上はmysqlのモジュールがなくなったらしく、拡張モジュールのphp-mysqlを入れないと、WordPressのセットアップ時に「対応していません」と表示されて進まなくなる。
これはmariadbでも同じみたい。
```
sudo apt install php-fpm php-mysql
```
3. phpのバージョンを確認する
```
sudo php -v
```
4. nginxのVirtualHost設定ファイルを作成する
VirtualHostとは？となったので、以下のサイトから勉強した。
https://developers.gmo.jp/technology/39513/
1台のサーバーで複数のドメインを運用するために必要なサーバー技術のこと。
ここはもう少しちゃんと理解しないといけないと思うが、とにかく今は進めてみる。

```
sudo vim /etc/nginx/sites-available/wordpress.conf
```
5. 先に作成したwordpressの構成ファイルに設定を投入
client max body size は色んなサイトで記載が違ったが、50M~100M程度でよさそう。
server_name にグローバルIPやドメイン名を入れないとアクセスできないので注意。複数入力の場合はスペース空けでOK。
```
server {
    listen 80;
    root /var/www/html;
    index index.php index.html index.htm;
    server_name XXX.XXX.XXX.XXX www.example.com;

    client_max_body_size 100M;
    autoindex off;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
         include snippets/fastcgi-php.conf;
         fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
         fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
         include fastcgi_params;
    }
}
```
6. 有効なサイトとするため、site-enebledにシンボリックリンクを作成
```
sudo ln -s /etc/nginx/sites-available/wordpress.conf /etc/nginx/sites-enabled/
```
7. nginxの再起動
```
sudo systemctl restart nginx
```

## WordPressの初期セットアップ
WordPressの圧縮ファイルを引っ張ってきてディレクトリに格納する。
1. /var/www/html に移動する
```
cd /var/www/html
```
2. WordPressに関連するファイルをダウンロードする
```
sudo wget https://ja.wordpress.org/latest-ja.tar.gz
```
3. ダウンロードしたファイルを展開する
```
sudo tar xvf latest-ja.tar.gz
```
4. wordpressファイルの所有者を変更
```
sudo chown -R www-data:www-data wordpress
```
あとは追加でwordpressで使うデータベースを作成しておく。
5. mariadbのコンソールに入る
```
sudo mariadb
```
6. データベースwordpressの作成（復習）
```
MariaDB [(none)]> CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8;
Query OK, 1 row affected (0.627 sec)
```
7. ユーザーの権限付与と有効化
```
MariaDB [(none)]> GRANT ALL ON wordpress.* TO 'さっき作ったユーザー名'@'localhost' IDENTIFIED BY 'yusaku';
Query OK, 0 rows affected (0.466 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
```

## ブラウザからWordPressのセットアップ
ここからは実際にブラウザへアクセスし、データベースの紐づけや初期セットアップを行う。
1. ブラウザからwordpressにアクセスする
```
http://<IPアドレス>/wordpress
```
![](https://storage.googleapis.com/zenn-user-upload/e46fdd4e0245-20250321.png)

2. ここまで作成したデータベースやユーザー名、パスワードを入れて進める
![](https://storage.googleapis.com/zenn-user-upload/44a1347f76ea-20250321.png)

3. データベースとの連携ができたら進める
![](https://storage.googleapis.com/zenn-user-upload/8f1b38953a83-20250321.png)

4. 必要な情報を入力してセットアップを完了する
![](https://storage.googleapis.com/zenn-user-upload/11dc2d3c3925-20250321.png)


## まとめ
記事を参考にさせてもらいながら進めていたが、やはりどこか足りない部分が出てきていい勉強になった。
もう少しかみ砕いて書いても良かったかもしれないが、また同じようにセットアップするかも…
まずはSSL化を次にやって、公開まで進めて行けたらいいし、
そろそろブログにするのかZennでこのまま書くのか、検討していこう。
ポートフォリオもやっていきましょ。

## 参考文献
https://noauto-nolife.com/post/ubuntu-add-php-repository/
https://note.com/frogspoon/n/n219381b003fb
