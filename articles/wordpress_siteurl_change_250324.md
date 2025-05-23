---
title: "WordPressのサイトURLの変更"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
WordPress初心者なのでよく知らなかったが、ドメイン名でアクセスしてもIPアドレスが表示されてしまい、変えないとと思っていた。
以前にWordPressの管理ページからURLを変更したらアクセスできなくなり、詰むといういかにもなトラブルを起こしており、今回はその反省を生かして備忘録にしておく。

## 方法はいくつかあるらしいが…
データベース側から変更をしたり、GUIとCUI両方で適用する方法があるらしいが、今回はwp-config.phpをそのまま変更することにした。
（一番簡単そうだったので）

## 変更手順
1. まずはWordPressのファイルが入っているディレクトリに移動
```
$ cd <ディレクトリ>
```
2. 中にあるwp-config.phpを編集する
```
$ sudo vi wp-config.php
```
3. DB_NAMEの前あたりに、以下の内容を追記する
(xxxの部分は自分のドメイン名を入力)
```
define( 'WP_HOME', 'http://xxx.com' );
define( 'WP_SITEURL', 'http://xxx.com' );
```
4. nginxのサービス再起動
```
$ sudo systemctl restart nginx
```

## 変更すると
今までアクセスした際にIPアドレスが表示されていたが、想定通りドメイン名が表示される形でアクセスすることができた。

ちなみに管理ページに移動すると、グレーアウトされて変更できなくなっている。

## まとめ
前回はもう諦めてリセットしてしまったのだが、簡単に変更できて反省。
次はSSL化、そのあとにサインインページのURLを変更したいと思う。