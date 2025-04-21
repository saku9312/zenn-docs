---
title: "CiscoのOSPF設定（別エリアを含む）"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに
実際の業務で使ったことはほとんどないが、CCNAの範囲だったOSPFをもう一度復習する。
前回はエリア内の設定を復習したので、次は異なるエリアが存在する場合の確認をする。

## 環境
- PacketTracer

## ネットワーク図
ルーターの829を使用し、OSPFで異なるエリアの情報を含めて伝達する。
お互いローカルに持っているVLANは101-105、これらもOSPFで伝達する。
![](https://storage.googleapis.com/zenn-user-upload/9fe841ad8079-20250421.png)

## 手順
例として、エリア0と1の境界ルーター（ABR）であるRouter0の設定を記載する。

1. 各ルーターでVLANを作成する。今回はVTPを使わない。
```
(config)# vlan 201
```
2. VLANに対してIPアドレスを付与し、インターフェイスに適用する
```
(config)# interface vlan 201
(config-vlan)# ip address 1.1.1.1 255.255.255.0
(config-if)# trunk系の設定
```
3. OSPFの有効化を行い、ネットワークとエリアを指定
```
(config)# router ospf 1
(config-router)# network 1.1.1.0 0.0.0.15 area 1
(config-router)# network 2.2.2.0 0.0.0.15 area 0
(config-router)# network 192.168.2.0 0.0.0.255 area 0
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
C       1.1.1.0/28 is directly connected, Vlan201
L       1.1.1.1/32 is directly connected, Vlan201
     2.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       2.2.2.0/28 is directly connected, Vlan202
L       2.2.2.2/32 is directly connected, Vlan202
     3.0.0.0/28 is subnetted, 1 subnets
O       3.3.3.0/28 [110/2] via 2.2.2.1, 00:11:06, Vlan202
     4.0.0.0/28 is subnetted, 1 subnets
O IA    4.4.4.0/28 [110/3] via 2.2.2.1, 00:10:56, Vlan202
O    192.168.1.0/24 [110/2] via 1.1.1.2, 00:14:44, Vlan201
     192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.2.0/24 is directly connected, Vlan102
L       192.168.2.1/32 is directly connected, Vlan102
O    192.168.3.0/24 [110/2] via 2.2.2.1, 00:16:50, Vlan202
O    192.168.4.0/24 [110/3] via 2.2.2.1, 00:10:56, Vlan202
O IA 192.168.5.0/24 [110/4] via 2.2.2.1, 00:08:26, Vlan202
```

## 忘れていたこと
- OSPFのエリア外のルートをテーブルに乗せると、O IAという表示になる。
今回はエリア2の情報が別エリアに関するルートになるため、その部分は「O IA」と表記される。
「O」はエリア内、「O IA」はエリア外、また「O E1」は外部ルートになる。

- セキュリティを担保するには、認証を含めることが必要。

## 次にやること
とりあえずエリア外の確認もできたので、次は認証とBGPの設定を行ってみたいと思う。