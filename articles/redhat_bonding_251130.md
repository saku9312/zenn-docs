---
title: "Red Hatでネットワークインターフェースのボンディング設定"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに
RedHatを構築中のミニPCがNICを2個持っているため、ボンディング設定を行って冗長化してみる。

## 構成
- Red Hat Linux ENterprise 10.0
- nmcli

## ボンディング設定の実施
1. ボンディングのインターフェースを作成する。
```
$ nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=1,primary=eth0,miimon=100"
```
2. IPアドレスやゲートウェイを設定する。
```
$ nmcli con mod bond0 ipv4.address "172.23.1.XXX/24"
$ nmcli con mod bond0 ipv4.method manual
$ nmcli con mod bond0 ipv4.gateway "172.23.1.254"
$ nmcli con mod bond0 connection.autoconnect yes
$ nmcli con mod bond0 ipv6.method disable
```
3. ボンディングインターフェースに、物理インターフェースを追加する。
```
$ nmcli con add type ethernet slave-type bond con-name bond0-port1 ifname eth0 master bond0
$ nmcli con add type ethernet slave-type bond con-name bond0-port2 ifname eth1 master bond0
```

# まとめ
ボンディング自体は簡単にできる。
実際は物理インターフェースにもDHCPでIPが振られていたため、その部分は停止する必要がある。