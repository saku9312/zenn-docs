---
title: "OSPFの動作をあらためてまとめる"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに
実際の業務で使ったことはほとんどないが、CCNAの範囲だったOSPFをもう一度復習する。
久々で忘れているところもあったのでたまに振り返るようにしたい。

## 環境
- PacketTracer

## ネットワーク図
ルーターの829を使用し、OSPFで同エリア内の情報を伝達する。
お互いローカルに持っているVLAN10、20、30もOSPFで伝達する。
![](https://storage.googleapis.com/zenn-user-upload/d6179454f972-20250415.png)

## 手順
1. 各ルーターでVLANを作成する。今回はVTPを使わない。
```
(config)# vlan 10
```
2. VLANに対してIPアドレスを付与し、インターフェイスに適用する
```
(config)# interface vlan 10
(config-vlan)# ip address 1.1.1.1 255.255.255.0
(config-if)# trunk系の設定
```
3. OSPFの有効化を行い、ネットワークとエリアを指定
```
(config)# router ospf 1
(config-router)# network 1.1.1.0 0.0.0.255 area 0
(config-router)# network 10.1.1.0 0.0.0.15 area 0
```
4. 各機器のルーティングテーブルやPacketTracerのシミュレーションでLSA配布の確認をする
```
# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     1.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       1.1.1.0/24 is directly connected, Vlan10
L       1.1.1.1/32 is directly connected, Vlan10
     2.0.0.0/24 is subnetted, 1 subnets
O       2.2.2.0/24 [110/2] via 10.1.1.2, 00:19:10, Vlan101
     3.0.0.0/24 is subnetted, 1 subnets
O       3.3.3.0/24 [110/3] via 10.1.1.2, 00:18:48, Vlan101
     10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
C       10.1.1.0/28 is directly connected, Vlan101
L       10.1.1.1/32 is directly connected, Vlan101
O       10.1.2.0/28 [110/2] via 10.1.1.2, 00:26:03, Vlan101
```

## 忘れていたこと
- OSPFのネットワークを指定しないと、他のルーターに情報を伝達しない
最初に各機器ローカルのVLANを指定しなくて、ルーティングテーブルに表示されてこなかった。OSPFで有効化するネットワークはすべて有効化しておく。

- ネットワークの指定時にエリア情報を最後に入れず、エラーを返された
ネットワーク有効化をする際は、エリアの情報を入れて進める。

## 次にやること
今回はすべて同じエリアで設定したが、実際はエリア0をバックボーンとして、エリアを分けることでルーティングテーブルを肥大化させない設定になると思う。
この後に続けてエリアを分けて設定し、実際のルーティングテーブルを確認してみる。