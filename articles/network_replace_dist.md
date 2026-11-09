---
title: "自宅回線を落とさない！ONU直下分岐によるIPoE（検証）× PPPoE（家庭用）完全分離ネットワーク構築"
emoji: "🌐"
type: "tech"
topics: ["network", "cisco", "ospf", "ipv6", "インフラ", "ccnp"]
published: true
---

# 1. はじめに

インフラエンジニアやネットワークエンジニアとしてスキルアップを目指す際、避けて通れないのが「自宅ラボ環境の構築」です。GNS3やCisco Modeling Labs（CML）などの仮想シミュレータも進化していますが、実トラフィックを流し、物理ケーブルを抜き差ししてフェイルオーバーを確認する「実機検証」には、シミュレータでは得られない圧倒的な学びがあります。

しかし、自宅で実機ルータやサーバ群を運用しようとすると、必ず以下の**「自宅ラボのジレンマ」**に直面します。

1. **生活回線との共存リスク:** 検証でOSPFやBGPの経路再配送を試したり、ルータを再起動したりした瞬間に、家族が使っているWi-Fiや動画配信が途切れて重大な家庭内インシデントに発展する。
2. **ルーティングとセグメントのカオス化:** 既存の家庭用Wi-Fiルータ配下に検証環境をぶら下げると、二重NAT、ポート開放の制限、意図しないDHCPパケットの漏洩（Rogue DHCP）に悩まされる。
3. **帯域の奪い合い:** 検証用サーバ（Proxmoxクラスタやバックアップ通信など）の大容量通信が、家庭用のWeb会議やゲームのトラフィックを圧迫する。

これらの課題を根本から解決するため、筆者は**「1本の光回線（ONU）をスイッチングハブで物理分岐させ、家庭用ネットワーク（PPPoE）と検証用エンタープライズネットワーク（IPoE）を物理・論理レベルで完全に二重化・分離するアーキテクチャ」**を設計・構築しました。

本記事では、このハイブリッドネットワークの設計思想から、構築最大の障壁となった**「NTT NGN網におけるIPv6プレフィックスの椅子取りゲーム問題」の技術的解決策**、CiscoとNECのマルチベンダ環境におけるOSPF・HSRP設計、そしてそのまま投入できる完全な実機コンフィグまで、実務・資格試験（CCNP ENCOR/ENARSI、ネットワークスペシャリスト）レベルの技術的背景を交えて徹底的に解説します。

---

# 2. 全体アーキテクチャとセグメント設計

## 2.1. 要件定義

本構成を設計するにあたり、以下の要件を設定しました。

* **完全な耐障害性分離:** 検証側ルータ（Cisco/NEC）が全停止・ループしても、家庭用Wi-Fiネットワーク（192.168.24.0/24）には1ミリ秒の通信断も発生させない。
* **回線特性に応じた役割分担:**
  * **家庭用ネットワーク:** 既存のPPPoE接続を収容し、安定した生活用Wi-Fi基盤とする。
  * **検証用ネットワーク:** 高速・大容量なIPoE（IPv4 over IPv6 / transix等）を収容し、サーバ群やCCNP演習の主軸とする。
* **エンタープライズ3層階層型モデルの踏襲:** Core層、Distribution層、Access層を実機で再現し、動的ルーティング（OSPF）とゲートウェイ冗長（HSRP）、L2冗長（LACP / RSTP）を完全実装する。

## 2.2. IPアドレッシングとVLAN設計

ネットワーク全体のIPアドレス体系は以下のように定義しました。

| レイヤー / セグメント | ネットワークアドレス | VLAN ID | デフォルトGW | 主な収容機器・用途 |
| :--- | :--- | :---: | :--- | :--- |
| **家庭用生活網** | `192.168.24.0/24` | - | `192.168.24.1` | 市販Wi-Fiルータ、家族のスマホ、家電、PC |
| **検証 Transit** | `172.23.99.0/29` | - | なし (L3直結) | IX2106 (.1), C1111 (.2), C891FJ (.3) のOSPF交換用 |
| **検証 VLAN 2 (有線)** | `172.23.1.0/24` | 2 | `172.23.1.254` (HSRP) | Proxmox 3ノード、DNS/DHCPミニPC、ラズパイ群、管理SVI |
| **検証 VLAN 10 (無線)** | `172.23.10.0/24` | 10 | `172.23.10.254` (HSRP) | 検証用Wi-Fi AP、検証端末 |

## 2.3. ネットワークトポロジ図

ONUの直下に5ポートのスイッチングハブを配置し、物理レイヤー（L1/L2）で家庭用と検証用を二股に分岐させています。

```text
                           [ インターネット ]
                                  │
                           [ NTT NGN網 ]
                                  │
                               [ ONU ]
                                  │
                 ┌────────────────┴────────────────┐
                 │    ONU直下 スイッチングハブ      │
                 └───────┬──────────────────┬──────┘
                         │                  │
        (EtherType: 0x8863/0x8864)          │ (EtherType: 0x86dd)
         PPPoEセッション (生活用)            │ IPoEネイティブ (検証用)
                         │                  │
            ┌────────────┴────────┐ ┌───────┴────────┐
            │   家庭用Wi-Fiルータ  │ │  NEC IX2106    │ 【Core層 / WAN Gateway】
            │   (PPPoE 終端)      │ │  (IPoE 終端)   │
            └────────────┬────────┘ └───────┬────────┘
                         │                  │ GE1.0 (172.23.99.1/29)
           192.168.24.0/24                  │
           (完全独立・生活網)                │ [ Transit: OSPF Area 0 ]
                                  ┌─────────┴─────────┐
                           GE0/0/0│172.23.99.2         │GE0: 172.23.99.3
                    ┌─────────────┴──────┐     ┌──────┴────────────┐
                    │ Cisco C1111-8P     │     │ Cisco C891FJ      │ 【Distribution層】
                    │ (HSRP Active / .252│     │ (HSRP Standby/.253│ (L3ルーティング & FHRP)
                    └──────┬──────┬──────┘     └──────┬──────┬─────┘
                           │      │                   │      │
                           │      └─────────┐ ┌───────┘      │ (Trunk: VLAN 2, 10)
                           │                │ │              │ (L2クロス・メッシュ接続)
                    ┌──────┴──────┐   ┌─────┴─┴─────┐        │
                    │ 2960X #1    │═══│ 2960X #2    │────────┘ 【Access層】
                    │(Root Bridge)│Po1│(Secondary)  │          (LACP / Rapid-PVST+)
                    └──────┬──────┘   └──────┬──────┘
                           │                 │
              ┌────────────┴─────────────────┴────────────┐
              │                                           │
       [ VLAN 2: 172.23.1.0/24 ]             [ VLAN 10: 172.23.10.0/24 ]
       ・Proxmox 3ノードクラスタ              ・検証用Wi-Fi AP
       ・常設DNS / DHCP / NTP (ミニPC)        ・検証用クライアントPC
       ・管理用ラズパイ群
```

---

# 3. 【最重要】ONU直下分岐における「IPv6プレフィックス奪い合い」問題

ONU直下にハブを置いて複数台のルータをぶら下げる構成を組む際、ネットワークスペシャリストやCCNPの知識を持っていても高確率でハマる落とし穴が存在します。それが**「NTT NGN網におけるIPv6プレフィックスの排他仕様」**です。

## 3.1. なぜIPoE接続が突然死するのか？

NTT東日本・西日本のNGN網（フレッツ光）では、回線の契約形態に応じて以下のいずれかの方式で宅内にIPv6プレフィックスを割り当てます。

1. **ひかり電話契約あり（DHCPv6-PD方式）:** NGN網からルータに対して `/56` のプレフィックスが委譲（Prefix Delegation）される。
2. **ひかり電話契約なし（RA方式）:** NGN網からマルチキャストで `/64` のプレフィックスがルータのWANポートに通知（Router Advertisement）される。

ここで決定的な問題となるのが、NGN網の設備（網終端装置やエッジルータ）が持つ**「1契約回線につき、最初にIPv6要求を出してきた1台の機器（同一DUID / MAC）にしかプレフィックスを払い出さない」**という排他制御仕様です。

もし、家庭用のルータ（PPPoE側）と検証用ルータ（IPoE側）の両方がWANポートでIPv6を有効化していると、**2台のルータ間で「IPv6アドレスの椅子取りゲーム」が発生**します。
PPPoEルータ側が先にNGNのIPv6プレフィックスを掴んでしまうと、本来IPoE（transix等の4in6トンネル）を張るべきIX2106側にはIPv6アドレスが一切降ってこなくなり、IPoEのトンネル確立が永久にタイムアウトします。

## 3.2. レイヤー2（EtherType）の視点で紐解く完全解決策

この問題を解決する鍵は、**「家庭用ルータ側で、IPv6のフレームを一切送受信させない（完全なIPv6盲目状態にする）」**ことです。

イーサネットフレームのレベルで見ると、PPPoEとIPoEは全く異なる言語（EtherType）を使用しています。

* **PPPoEの通信:**
  * **EtherType `0x8863`:** PPPoE Discovery（セッション開始前の探索）
  * **EtherType `0x8864`:** PPPoE Session（確立後のPPPカプセル化データ通信）
  * PPPoEは網内のBAS（Broadband Access Server）と1対1の仮想トンネルを張ってその中でIPv4を喋るため、外部のIPv6通信とは物理線を共有していても完全に直交しています。
* **IPoEの通信:**
  * **EtherType `0x86dd`:** IPv6 ネイティブパケット
  * NGN網のルータと直接IPv6パケットを送受信し、VNE事業者のAFTR（CGNAT）と4in6トンネルを形成します。

```text
                  [ NTT NGN網 ]
                        │
                     [ ONU ]
                        │
             [ L2 スイッチングハブ ]
             ┌──────────┴──────────┐
             │                     │
 (EtherType: 0x8863/0x8864)  (EtherType: 0x86dd のみ)
             │                     │
   [ 家庭用PPPoEルータ ]     [ 検証用Coreルータ (IX2106) ]
   ・IPv6を完全無効化        ・NGNから見て「宅内のIPv6機器は
   ・PPPoEトンネルのみ確立     このルータ1台のみ」に見える
                             ・100%確実にIPv6プレフィックスを取得
```

家庭用ルータのWANインターフェースにおいて、IPv6の自動設定（`ipv6 address autoconfig`）やIPv6スタック自体を無効化（`no ipv6 enable`）すれば、家庭用ルータはEtherType `0x86dd` のパケットを一切送信せず、受信しても即座にハードウェアで破棄（ドロップ）します。

これにより、NGN側から見た宅内のIPv6要求元は**「検証用ルータ（IX2106）ただ1台」**になり、アドレス競合や奪い合いが物理的・論理的に100%発生しなくなります。

---

# 4. Core-Distribution層のL3動的ルーティング設計（Cisco × NEC）

検証ネットワークの内部では、WANエッジであるNEC IX2106と、Distribution層である2台のCiscoルータ（C1111 / C891FJ）との間で**OSPF（Open Shortest Path First）Area 0**を回します。

単一ベンダであれば初期設定のままでもネイバーが上がりますが、**Cisco（IOS / IOS XE）とNEC（UNIVERGE IX）のマルチベンダ環境では、プロトコルの仕様解釈の違いから障害が多発**します。CCNP試験や実務設計で必須となる2大ポイントを解説します。

## 4.1. MTU不一致による「EXSTART / EXCHANGE」スタック問題

OSPFのネイバー確立プロセスでは、Down → Init → 2-Way を経て、DBD（Database Description）パケットを交換する **ExStart / Exchange** ステートに遷移します。

このとき、OSPFヘッダ内の「Interface MTU」フィールドがチェックされますが、**対向ルータ間でMTU値が1バイトでも食い違っていると、ルータはDBDパケットの受信を拒否し、ステートが `EXSTART` または `EXCHANGE` のまま停止（スタック）**します。

* **Ciscoの挙動:** デフォルトで自身のインターフェースMTU（通常1500）を厳密にチェックする。
* **NEC IXの挙動:** ファームウェアやインターフェースの種別（VLANタグ付与時など）によって、MTUの解釈が1500や1514などでズレることがある。

### 対策
対向するインターフェース（Transitセグメント: 172.23.99.0/29）において、両者のIP MTUが確実に `1500` であることを設定・確認します。Cisco側で `ip ospf mtu-ignore` を打つことで強制無視させる回避策もありますが、LSA（リンクステート情報）が肥大化した際にパケットが断片化・破棄される原因となるため、**実インターフェースのMTUを綺麗に揃えるのが設計の正道**です。

## 4.2. OSPFコスト計算基準帯域（Reference Bandwidth）の不整合

OSPFのメトリック（Cost）は、以下の計算式で算出されます。

$$\text{Cost} = \frac{\text{Reference Bandwidth (基準帯域)}}{\text{Interface Bandwidth (インターフェース実効帯域)}}$$

この「基準帯域」のデフォルト値が、ベンダやOSの世代によって大きく異なります。

* **Cisco IOS / IOS XE のデフォルト:** **100 Mbps** ($10^8$ bps)
  * 100Mbpsのリンク: $\frac{100M}{100M} = \text{Cost } 1$
  * 1Gbps（1000Mbps）のリンク: $\frac{100M}{1000M} = 0.1 \rightarrow \text{Cost } 1$（切り上げ）
  * **問題点:** 100Mbpsも1Gbpsも10Gbpsも、すべてコストが「1」になってしまい、帯域の太さを正しく評価できない。
* **NEC UNIVERGE IX のデフォルト:** 機種や世代によって **1 Gbps** ($10^9$ bps) またはそれ以上

基準帯域が異なると、Cisco側とNEC側で計算されるルートのメトリックがチグハグになり、**「行きはC1111を通ったのに、帰りはC891FJを通る」といった非対称ルーティング（Asymmetric Routing）**や、低速リンクへの誤った迂回が発生します。

### 対策
Cisco側のOSPFプロセス配下で必ず `auto-cost reference-bandwidth` を設定し、ネットワーク全体の基準帯域を統一します。

```text
! 1Gbps (1000Mbps) を基準帯域に設定する場合
router ospf 1
 auto-cost reference-bandwidth 1000
```

## 4.3. デフォルトルートの広報と内部ネットワークの保護

* **Coreルータ（IX2106）:** インターネットへのデフォルトルートを持っているため、OSPF内に `default-information originate always` を設定し、配下のCiscoルータへデフォルトルート（`O*E2 0.0.0.0/0`）を一括配信します。
* **Distributionルータ（C1111 / C891FJ）:** 内部セグメント（VLAN 2: `172.23.1.0/24`、VLAN 10: `172.23.10.0/24`）をOSPFに広告し、外から戻ってきたパケットが正確に下位へ届くようにします。
* **Passive-Interfaceの徹底:** サーバやPCが接続される内部LANインターフェース（SVI Vlan2, Vlan10）に対して、不要なOSPF Helloパケットを送出しないよう `passive-interface` を設定します（CCNPセキュリティ・ベストプラクティス）。

---

# 5. Distribution層におけるデフォルトゲートウェイ冗長化（HSRP）

配下のサーバ群（Proxmox、DNS、Zabbix等）やWi-Fiクライアントにとって、デフォルトゲートウェイの停止は完全なネットワーク切断を意味します。これを防ぐため、2台のCiscoルータ（C1111とC891FJ）間で **HSRP（Hot Standby Router Protocol）バージョン2** を稼働させます。

## 5.1. HSRPv2設計パラメータ

* **仮想IPアドレス（VIP）:**
  * VLAN 2: `172.23.1.254`
  * VLAN 10: `172.23.10.254`
* **優先度（Priority）と役割分担:**
  * **Cisco C1111-8P:** `Priority 110` ＋ `Preempt` 有効 → **【Activeルータ】**
  * **Cisco C891FJ:** `Priority 100` (デフォルト) → **【Standbyルータ】**

高性能なC1111（IOS XE）を通常時の通信担当（Active）とし、C1111の障害時やメンテナンス時のみ、C891FJ（Classic IOS）が自動的に仮想IPと仮想MACアドレスを引き継ぎます。`preempt` を有効にしているため、C1111の復旧後は自動的にActiveへ切り戻されます。

## 5.2. Accessスイッチ（Catalyst 2960X）とのRapid-PVST+連携

Distributionルータ2台とAccessスイッチ2台（Catalyst 2960X #1, #2）は、耐障害性を極限まで高めるために**クロス接続（L2メッシュトポロジ）**で配線されています。

ここで重要になるのが、**「STPルートブリッジ」と「HSRP Activeルータ」の位置の同期**です（CCNP ENCOR設計分野の鉄板項目）。

* **不整合のアンチパターン:**
  もしSTPルートブリッジを「2960X #2」にし、HSRP Activeを「C1111（2960X #1直結）」にしてしまうと、端末から出たフレームが「スイッチ間リンクを渡ってからルータへ向かう」という無駄な迂回（トロンボーン現象）が発生します。
* **最適化設計:**
  C1111が直結されている **Catalyst 2960X #1 をSTP Primary Root Bridge（Priority 4096）** に固定し、最短経路でActiveルータへパケットが抜けるようにコストを最適化します。

---

# 6. 実機完全コンフィグ集

本環境で稼働している各機器の完全コンフィグ（設定投入用テキスト）です。コピー＆ペーストしてそのまま実機に投入できるように整形しています。

## 6.1. 家庭用ルータ（PPPoE専用・IPv6完全無効化）

NGN網のIPv6を奪わないよう、WAN側でIPv6を完全に遮断し、IPv4 PPPoEのみを終端する設定です（Ciscoルータを家庭用にする場合の例）。

```text
enable
configure terminal

hostname Home-GW

! --- WAN側物理ポート（IPv6の完全無効化） ---
interface GigabitEthernet0
 description WAN to ONU (PPPoE Only)
 no ip address
 no ipv6 enable
 no ipv6 address autoconfig
 no ipv6 nd autoconfig default-route
 pppoe enable group global
 pppoe-client dial-pool-number 1
 no shutdown
exit

! --- PPPoE論理インターフェース ---
interface Dialer1
 description PPPoE to ISP (IPv4 Home)
 mtu 1454
 ip address negotiated
 ip nat outside
 encapsulation ppp
 dialer pool 1
 dialer-group 1
 no ipv6 enable
 ! プロバイダ認証（PAP/CHAP両対応設定）
 ppp chap hostname <家庭用プロバイダID>
 ppp chap password 0 <家庭用パスワード>
 ppp pap sent-username <家庭用プロバイダID> password 0 <家庭用パスワード>
 ip tcp adjust-mss 1414
 no shutdown
exit

! --- 家庭用LANポート (192.168.24.0/24) ---
interface Vlan1
 description Home_LAN_WiFi
 ip address 192.168.24.1 255.255.255.0
 ip nat inside
 ip tcp adjust-mss 1414
 no shutdown
exit

! --- NAT / デフォルトルート ---
ip route 0.0.0.0 0.0.0.0 Dialer1
access-list 1 permit 192.168.24.0 0.0.0.255
ip nat inside source list 1 interface Dialer1 overload

end
copy run start
```

## 6.2. Coreルータ：NEC UNIVERGE IX2106（IPoE / OSPF Area 0）

インターネット側をIPoEで終端し、TransitセグメントでCiscoルータ群とOSPFネイバーを張ってデフォルトルートを配布します。

```text
enable-config

hostname IX2106

! --- WANポート（GE0: IPoE / transix等の設定を適用） ---
interface GigaEthernet0.0
  no shutdown
exit

! --- Transit LANポート（Cisco対向ルーテッドポート） ---
interface GigaEthernet1.0
  ip address 172.23.99.1/29
  ip tcp adjust-mss 1414
  no shutdown
exit

! --- OSPF Area 0 設定 ---
router ospf 1
  router-id 172.23.99.1
  network 172.23.99.0/29 area 0.0.0.0
  ! 下位のCiscoルータへデフォルトルートを広告
  default-information originate always
exit

exit
write memory
```

## 6.3. Distributionルータ #1：Cisco C1111-8P（OSPF / HSRP Active）

内部VLANのコア集約、OSPFルート広告、およびHSRP Activeとして動作する主軸ルータです。

```text
enable
configure terminal

hostname C1111
ip routing

! --- VLAN定義 ---
vlan 2
 name Server_Core
!
vlan 10
 name Client_WiFi
exit

! --- Transitポート（IX2106対向ルーテッドポート） ---
interface GigabitEthernet0/0/0
 description Link to IX2106 (Transit L3)
 no switchport
 ip address 172.23.99.2 255.255.255.248
 ip tcp adjust-mss 1414
 no shutdown
exit

! --- SVI: VLAN 2 (有線・サーバ用 / HSRP Active) ---
interface Vlan2
 description Wired_Servers_Core
 ip address 172.23.1.252 255.255.255.0
 ip tcp adjust-mss 1414
 standby version 2
 standby 2 ip 172.23.1.254
 standby 2 priority 110
 standby 2 preempt
 no shutdown
exit

! --- SVI: VLAN 10 (無線・クライアント用 / HSRP Active) ---
interface Vlan10
 description Wireless_Clients
 ip address 172.23.10.252 255.255.255.0
 ip tcp adjust-mss 1414
 standby version 2
 standby 10 ip 172.23.10.254
 standby 10 priority 110
 standby 10 preempt
 no shutdown
exit

! --- Catalyst 2960X 対向トランクポート ---
interface GigabitEthernet0/1/0
 description Trunk to Catalyst-2960X-1
 switchport mode trunk
 switchport trunk allowed vlan 2,10
 no shutdown
exit

interface GigabitEthernet0/1/1
 description Trunk to Catalyst-2960X-2
 switchport mode trunk
 switchport trunk allowed vlan 2,10
 no shutdown
exit

! --- OSPF Area 0 設定 ---
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 1000
 ! Transitリンクでネイバー確立
 network 172.23.99.0 0.0.0.7 area 0
 ! 内部セグメントをIX2106へ広報
 network 172.23.1.0 0.0.0.255 area 0
 network 172.23.10.0 0.0.0.255 area 0
 ! 内部LANへの不要なHelloパケット抑止
 passive-interface Vlan2
 passive-interface Vlan10
exit

end
copy run start
```

## 6.4. Distributionルータ #2：Cisco C891FJ（OSPF / HSRP Standby）

C1111の対胴として待機し、障害発生時に無瞬断でゲートウェイを引き継ぐスタンバイルータです。

```text
enable
configure terminal

hostname C891FJ
ip routing

! --- VLAN定義 ---
vlan 2
 name Server_Core
!
vlan 10
 name Client_WiFi
exit

! --- Transitポート（IX2106対向ルーテッドポート） ---
interface GigabitEthernet 0
 description Link to IX2106 (Transit L3)
 no switchport
 ip address 172.23.99.3 255.255.255.248
 ip tcp adjust-mss 1414
 no shutdown
exit

! --- SVI: VLAN 2 (有線・サーバ用 / HSRP Standby) ---
interface Vlan2
 description Wired_Servers_Core
 ip address 172.23.1.253 255.255.255.0
 ip tcp adjust-mss 1414
 standby version 2
 standby 2 ip 172.23.1.254
 standby 2 priority 100
 no shutdown
exit

! --- SVI: VLAN 10 (無線・クライアント用 / HSRP Standby) ---
interface Vlan10
 description Wireless_Clients
 ip address 172.23.10.253 255.255.255.0
 ip tcp adjust-mss 1414
 standby version 2
 standby 10 ip 172.23.10.254
 standby 10 priority 100
 no shutdown
exit

! --- Catalyst 2960X 対向トランクポート（内蔵スイッチポート） ---
interface FastEthernet 0
 description Trunk to Catalyst-2960X-1
 switchport mode trunk
 switchport trunk allowed vlan 2,10
 no shutdown
exit

interface FastEthernet 1
 description Trunk to Catalyst-2960X-2
 switchport mode trunk
 switchport trunk allowed vlan 2,10
 no shutdown
exit

! --- OSPF Area 0 設定 ---
router ospf 1
 router-id 2.2.2.2
 auto-cost reference-bandwidth 1000
 network 172.23.99.0 0.0.0.7 area 0
 network 172.23.1.0 0.0.0.255 area 0
 network 172.23.10.0 0.0.0.255 area 0
 passive-interface Vlan2
 passive-interface Vlan10
exit

end
copy run start
```

## 6.5. Accessスイッチ：Cisco Catalyst 2960X（LACP / RSTP）

スイッチ間の2本をLACPで束ね、Rapid-PVST+でループを制御しつつ、エッジポートを保護します。

```text
! =========================================================
! Catalyst 2960X #1 (Primary Root Bridge)
! =========================================================
enable
configure terminal

hostname Switch-2960X-1

! --- スパニングツリー設定（Primary Root） ---
spanning-tree mode rapid-pvst
spanning-tree vlan 2,10 priority 4096

! --- スイッチ間 EtherChannel (LACP) ---
interface range GigabitEthernet0/9 - 10
 description Link to Switch-2960X-2 (LACP)
 switchport mode trunk
 switchport trunk allowed vlan 2,10
 channel-group 1 mode active
 no shutdown
exit

! --- 上位ルータ（C1111 / C891FJ）対向ポート ---
interface GigabitEthernet0/1
 description Uplink to C1111
 switchport mode trunk
 switchport trunk allowed vlan 2,10
 spanning-tree guard root
 no shutdown
exit

interface GigabitEthernet0/2
 description Uplink to C891FJ
 switchport mode trunk
 switchport trunk allowed vlan 2,10
 spanning-tree guard root
 no shutdown
exit

! --- サーバ・端末収容アクセスポート（PortFast & BPDU Guard） ---
interface range GigabitEthernet0/3 - 8
 switchport mode access
 switchport access vlan 2
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

end
copy run start
```

---

# 7. ダウンタイム数秒で完結する安全な切り替え手順（SOP）

本番稼働中の環境でネットワーク機器を入れ替える際、最もやってはならないのが「ぶっつけ本番で配線を挿し替え、コンソールで手動設定を打ち込むこと」です。トラブルが発生した瞬間に何が原因か切り分けられなくなります。

今回は以下の**「5段階の段階的移行（フェーズアプローチ）」**を策定して実行しました。

## Phase 1: 机上事前設定（完全オフライン）
新設する機器（IX2106、C1111）を検証デスクに並べ、PCと1対1でコンソール接続して、上記の設定をすべて投入・保存（`write memory`）しておきます。この段階では本番のLANケーブルは1本も接続しません。

## Phase 2: L2スイッチ基盤のSTP保護先行投入
稼働中のCatalyst 2960Xに対し、通信影響のない範囲で `spanning-tree mode rapid-pvst` や `bpduguard enable` などの保護コマンドを先行投入しておきます。これにより、後からルータのメッシュ配線を接続した際の意図しないブロードキャストストームを未然に防ぎます。

## Phase 3: TransitセグメントとOSPFの開通
IX2106とC1111をLANケーブルで直結し、机上でOSPFネイバーが `FULL` になること、およびデフォルトルートが降ってくることを確認します。

## Phase 4: ゲートウェイのスライド切り替え（通信断：数秒）
ここが唯一の切り替えポイントです。
1. C891FJのIPアドレスを一時的に書き換えて `.254` を解放する（例: `.253` に変更）。
2. C1111をスイッチへLANケーブルで接続する。
3. C1111は対向がいないため即座にHSRP `Active` となり、仮想IP（`.254`）のGratuitous ARPをブロードキャストします。
4. サーバ群や端末側のARPテーブルが更新され、通信がC1111経由へ一瞬でスライドします。

## Phase 5: WANケーブルの分岐と最終疎通
ONUからのLANケーブルを5ポートハブに挿し、家庭用ルータとIX2106へそれぞれ配線します。これで家庭用（PPPoE）と検証用（IPoE）の二重化が完了します。

---

# 8. 正常性確認コマンド集と期待される出力

切り替え完了後、以下のコマンドを実行して各プロトコルが健全に動作しているかを確認します。

### ① OSPFネイバーの確認（Cisco C1111側）
```bash
C1111# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.23.99.1       1   FULL/BDR        00:00:34    172.23.99.1     GigabitEthernet0/0/0
2.2.2.2           1   FULL/DR         00:00:31    172.23.99.3     GigabitEthernet0/0/0
```
* **確認ポイント:** 対向のIX2106（`172.23.99.1`）およびC891FJ（`2.2.2.2`）とのステートが **`FULL`** になっていること。`EXSTART` で止まっていないことを確認します。

### ② OSPFルーティングテーブルの確認（Cisco C1111側）
```bash
C1111# show ip route ospf

Gateway of last resort is 172.23.99.1 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 172.23.99.1, 00:15:22, GigabitEthernet0/0/0
```
* **確認ポイント:** IX2106から広告されたデフォルトルート（`O*E2 0.0.0.0/0`）がネクストホップ `172.23.99.1` としてルーティングテーブルに登録されていること。

### ③ HSRP冗長化ステータスの確認（Cisco C1111側）
```bash
C1111# show standby brief

                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl2         2    110 P Active  local           172.23.1.253    172.23.1.254
Vl10        10   110 P Active  local           172.23.10.253   172.23.10.254
```
* **確認ポイント:** C1111側で自身のステートが **`Active`**、対向のStandbyルータとしてC891FJ（`.253`）が認識され、Virtual IPが設計通りの `.254` になっていること。

---

# 9. おわりに＆今後の展望

本構成の実現により、**「どれだけ検証環境で大規模な設定変更や障害試験を行っても、家庭の生活用ネットワークには1ミリ秒の影響も与えない」**という、インフラエンジニアにとって究極のサンドボックス環境を手に入れることができました。

ネットワークの強固な足回りが完成したため、今後はGitHubリポジトリを活用した**「NetDevOps（GitOps）」**のフェーズへ進める予定です。

1. **Docs as Codeの推進:** IPアドレス管理（IPAM）やトポロジ図をMarkdownやdraw.ioでコード管理し、Gitで差分追跡を行う。
2. **自動化パイプラインの構築:** Cisco C1111（IOS XE）のRESTCONF/NETCONFを活用し、PythonスクリプトやAnsibleによるコンフィグの自動バックアップと整合性チェックを実装する。
3. **統合監視と認証統合:** Proxmoxクラスタ上にオープンソースSIEMである「Wazuh」をデプロイして全機器のSyslogを集約し、さらに「FreeIPA」を立ち上げてLinuxサーバ群の鍵管理・集中認証基盤を構築する。

物理環境特有の泥臭いトラブルシューティング（EtherTypeやMTU、プロトコルの癖）を乗り越えた経験は、そのまま現場での設計力や障害対応力に直結します。
自宅ラックやホームラボの構築に挑んでいるエンジニアの方々に、本記事が少しでも参考になれば幸いです！