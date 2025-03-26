---
title: "WordPressのSSL化、ログインページの秘匿"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
予定通り、自分のポートフォリオサイトをSSL化してみた。
Certbotのインストールがほとんどだが、一部サイトアドレスの変更も必要になった。

## SSL化の手順
SSL化はCertbotをインストールして、証明書を作成する形になる。
1. Certbotのインストール
```
$ sudo apt install certbot python3-certbot-nginx
```
2. /etc/nginx/sites-enabled/<公開ページ>.conf　を確認する
```
$ sudo vi /etc/nginx/sites-enabled/<公開ページ>.conf
    server {
    server_name www.xxx.jp;

    location / {
        root /home/<公開ページ>;
        index index.html;
    }
}
```
3. 証明書の発行
登録したいドメイン名を入力して、そのあとにメールアドレスを入力する。
```
$ sudo certbot --nginx -d ＜登録するドメイン名＞
```
実行すると、以下の内容が表示される。
```
www.xxx.jp
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): メールアドレスを入力

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.5-February-24-2025.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y
Account registered.
Requesting a certificate for www.xxx.jp

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/www.xxx.jp/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/www.xxx.jp/privkey.pem
This certificate expires on 2025-06-23.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for wwww.xxx.jp to /etc/nginx/sites-enabled/wordpress.conf
Congratulations! You have successfully enabled HTTPS on https://www.xxx.jp

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

```
4. nginxのサービス再起動
```
$ sudo systemctl restart nginx
```

## 証明書の定期更新
Let's Encrypt は3か月で証明書の期限が切れてしまうため、cronを使って定期的に更新をするように仕込む。

1. cron に設定を行う（rootで実行できるようにユーザー指定）
```
$ sudo crontab -u root -e
    # 以下の内容を追記(毎月初日の4時に実行、nginxの再起動)
    00 04 01 * * certbot renew && systemctl restart nginx
```
2. 設定内容の確認
```
$ sudo crontab -u root -l
```

これで証明書の更新が切れることにびくびくしなくても良くなった。

## サイトアドレスの変更
前回、WordPressにアクセスした際の表示アドレスを変更するため、wp-config.phpを修正した。
そのため、ここを変更しないと正常に表示されないので、変更しておく。

1. まずはWordPressのファイルが入っているディレクトリに移動
```
$ cd <ディレクトリ>
```
2. 中にあるwp-config.phpを編集する
```
$ sudo vi wp-config.php
```
3. DB_NAMEの前あたりに追記した内容を、httpからhttpsに変更する
(xxxの部分は自分のドメイン名を入力)
```
define( 'WP_HOME', 'https://xxx.com' );
define( 'WP_SITEURL', 'https://xxx.com' );
```
4. nginxのサービス再起動
```
$ sudo systemctl restart nginx
```

## ログインURLの変更
基本的にWordPressは/wp-admin や /wp-login でログインページにアクセスできるが、このままでは他のユーザーからログイン試行される可能性がある。

WPS Hide Login というプラグインをインストールすると簡単に変更ができるので、以下を試して反映しておいた。

https://kinsta.com/jp/blog/wordpress-login-url/

## まとめ
これでSSL化も完了したため、WordPressの環境構築に関してはほぼ終了した。
あとはバックアップを都度取得し、非常時に復旧できるように設定を追加したい。

## 参考ページ
https://qiita.com/shin4488/items/8738e13b92143c88aab6