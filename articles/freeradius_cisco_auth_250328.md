---
title: "freeRADIUSとCiscoスイッチを使用した認証設定"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
RADIUSサーバーってどうやって動かすの？
というのがずっと気になっていたので、実際の環境で認証ができるように設定してみた。
やることはかなりシンプルだったが、IOSのバージョンが最新だったことでネット上の記事ではうまくいかなかった。
そのあたりも含めて記載する。

## 検証環境
- Ubuntu 23.04(ProxmoxVE CT)
- Cisco Catalyst 2960CX-8TC-L(Version 15.2(4)E5)

## 検証手順
以下の内容で検証を開始する。
1. ProxmoxVEを使用してUbuntuのコンテナを作成し、freeRADIUSとufwをインストールする。
2. CiscoスイッチにRADIUS認証の設定を追加する。
3. TeratermからSSHでスイッチに接続し、認証情報として使用できるか検証する。

## freeRADIUSのセットアップ
まずはfreeRADIUSのセットアップから開始。
RADIUS認証時にはシークレットキーを揃える必要があるため、ネットワークの指定時にキーを決めておく。

1. UbuntuからfreeRADIUSのインストール
```
$ sudo apt-get install freeradius freeradius-utils
```
2. freeRADIUSのバージョンを確認
```
$ freeradius -v
radiusd: FreeRADIUS Version 3.0.26, for host aarch64-unknown-linux-gnu, built on Dec 17 2024 at 15:46:27
FreeRADIUS Version 3.0.26
Copyright (C) 1999-2021 The FreeRADIUS server project and contributors
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE
You may redistribute copies of FreeRADIUS under the terms of the
GNU General Public License
For more information about these matters, see the file named COPYRIGHT
```
3. 認証対象とするネットワークを指定するため、/etc/freeradius/3.0/clients.conf を編集する
```
$ cd /etc/freeradius/3.0
$ vi clients.conf
    #以下の部分を有効化もしくは追記
    client private-network-1 {
        ipaddr          = 0.0.0.0/0
        secret          = <シークレットキー>
```
4. ログインするユーザーを、/etc/freeradius/3.0/users に追記する
```
$ vi users
    #ユーザー情報を追記
    <ユーザー名> Cleartext-Password := "<パスワード>"
    ex) test-user-002 Cleartext-Password := "password123"
```
5. サービスの再起動
```
$ sudo systemctl enable freeradius
$ sudo systemctl restart freeradius
```

## Ciscoスイッチ側の設定
次はスイッチ側の設定を行う。
初めてaaa newmodelコマンドを実機に入れた気がする。
今回は優先的にサーバーのアカウントを使用、もし応答がなければもともと設定しているローカルアカウントを使用できるように設定。
username コマンドなどですでにローカルアカウントは作成しているものとする。

1. AAA認証の有効化
```
(config)# aaa new-model
```
2. RADIUSサーバーの登録
auth-portは認証・認可用のポートで、acct-portはアカウンティングのポートを指定できる。今回は1812,1813を使用。
シークレットキーにはサーバー側で設定したものを入力する。
```
(config)# radius server <サーバー名>
(config-radius-server)# address ipv4 <RADIUSサーバーのipアドレス> auth-port 1812 acct-port 1813
(config-radius-server)# key <シークレットキー>
```
4. RADIUSサーバーを認証サーバーとしてグループに登録
```
(config)# aaa group server radius <サーバーグループ名>
(config-sg-server)# aaa group server radius <サーバー名>
```
5. 認証認可、アカウンティングの際に使えるように設定
今回はログイン時の利用のみテストするため、一番上のものだけを投入。
```
(config)# aaa authentication login default group <サーバーグループ名> local
#(config)# aaa authentication dot1x default group <サーバーグループ名> local
#(config)# aaa authorization network default group <サーバーグループ名> local
#(config)# aaa authorization auth-proxy default group <サーバーグループ名> local
#(config)# aaa accounting dot1x default start-stop group <サーバーグループ名> local
#(config)# aaa accounting system default start-stop group <サーバーグループ名> local
```
6. 認証テスト
TeratermからSSHで接続してみる。問題がなければそのまま認証画面に進めるはず。

## 引っかかった点
- RADIUSサーバー
多くの記事でディレクトリパスが、/etc/raddb/ と書かれていたのだが、この環境では/etc/freeradius/3.0/だった。

- Ciscoスイッチ
IOSのバージョンが新しくなったため、
radius server host コマンドが使えなくなっており、
新しくグループに入れて投入することが必要になっていた。
https://www.cisco.com/c/ja_jp/support/docs/security-vpn/remote-authentication-dial-user-service-radius/200403-AAA-Server-Priority-explained-with-new-R.html

## まとめ
認証はできたので、また認可の部分もテストしてみたい。