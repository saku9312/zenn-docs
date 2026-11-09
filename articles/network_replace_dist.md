---
title: "自宅回線を落とさない！ONU直下分岐によるIPoE（生活用）× PPPoE（検証用IPsec-VPN）完全分離ネットワーク構築"
emoji: "🌐"
type: "tech"
topics: ["network", "cisco", "ospf", "ipsec", "ipv6", "ccnp"]
published: true
---

# 1. はじめに

インフラエンジニアやネットワークエンジニアとしてCCNP（ENCOR/ENARSI）やネットワークスペシャリストを目指す際、避けて通れないのが「自宅ラボ環境の構築」です。GNS3やCisco Modeling Labs（CML）などの仮想シミュレータも優秀ですが、実トラフィックを流し、物理ケーブルを抜き差ししてフェイルオーバーを確認する「実機検証」には、シミュレータでは得られない現場直結の経験値があります。

しかし、自宅で実機ルータやサーバ群を運用しようとすると、必ず以下の**「自宅ラボのジレンマ」**に直面します。

1. **生活回線との共存リスク:** 検証でOSPFやBGPの経路再配送を試したり、ルータを再起動したりした瞬間に、家族が使っているWi-Fiや動画配信が途切れて重大な家庭内インシデントに発展する。
2. **外部アクセスの制約（CGNATの壁）:** 近年主流のIPoE（IPv4 over IPv6 / transixやMAP-Eなど）は高速な反面、プロバイダ側のCGNAT（キャリアグレードNAT）によってポート開放や固定IPの利用が制限され、外部から自宅ルータへ直接着信するIPsec-VPN等の検証が極めて困難になる。
3. **帯域の奪い合い:** 検証用サーバ（Proxmoxクラスタやバックアップ通信など）の大容量通信が、家庭用のWeb会議やゲームのトラフィックを圧迫する。

これらの課題を根本から解決するため、本記事では**「1本の光回線（ONU）をスイッチングハブで物理分岐させ、家庭用ネットワーク（IPoE）と検証用エンタープライズネットワーク（PPPoE）を物理・論理レベルで完全に二重化・分離するアーキテクチャ」**の設計と構築手法を解説します。

検証用ルータ（NEC IX2106）にPPPoEを通すことで**本物のグローバルIPv4アドレスを直収し、外部PCからのIPsec-VPN着信検証を可能にする設計**や、**「NGN網におけるIPv6プレフィックスの奪い合い問題」の技術的解決策**、CiscoとNECのマルチベンダ環境におけるOSPF・HSRP設計、そしてそのまま投入できる完全な実機コンフィグまで網羅してまとめました。

---

# 2. なぜ「家庭＝IPoE」「検証＝PPPoE」の逆転構成なのか？

当初は「家庭用＝PPPoE、検証用＝IPoE」という構成も検討されますが、エンタープライズVPNやリモートアクセスの検証を本気で行う場合、**「家庭用＝IPoE、検証用＝PPPoE」にするのが圧倒的に合理的**です。

```text
               [ 光回線 (ONU) ]
                      │
           [ スイッチングハブ ]
           ／                ＼
         ／                    ＼
   [ 市販Wi-Fiルータ ]     [ NEC IX2106 (Core) ]
   【transix (IPoE)】      【PPPoE接続】
   ・家庭用・生活用ネット   ・検証用エンタープライズ網
   ・高速・高安定（混雑回避）・「グローバルIPv4」を直収
   ・ポート開放不要         ・外部からのIPsec-VPN着信が自由自在
```

## 2.1. IPoE（transix / DS-Lite）が抱える検証上の限界
家庭用通信として大普及しているtransix（DS-Lite）などのIPoE方式は、NGN網（IPv6）上にトンネルを掘ってVNE事業者の大型ルータ（AFTR）へパケットを運び、そこでまとめてIPv4 NAT（CGNAT）を掛けてインターネットへ抜けます。
この仕組み上、以下の制約が発生します。

* **ルータがグローバルIPv4アドレスを持たない:** ルータ自身に割り当てられるのはIPv6アドレスのみであり、外部からIPv4パケットを直接ルータ宛てに着信させることができない。
* **着信ポートの開放が不可:** ポート番号がプロバイダ側で動的に割り振られる、または塞がれているため、外部のPCやスマホから「自宅のルータ宛てにIPsecトンネル（UDP 500/4500、ESP）を張る」という検証が成立しない。

## 2.2. PPPoEでグローバルIPv4を直収するメリット
検証用ルータ（NEC IX2106）でPPPoEセッションを直接終端させると、ルータのWANインターフェースに**独立したグローバルIPv4アドレスが直接割り当てられます**。

* **外部からのIPsec-VPN直接着信:** 外出先のPC（Windows標準クライアントなど）やスマホから、自宅のIX2106宛てに直接トンネル（L2TP/IPsecやIKEv2 VPN）を張り、内部の検証サーバ群（ProxmoxやZabbixなど）へ安全にリモートアクセスする実務構成を検証できる。
* **NAT-T（NAT Traversal）やスプリットトンネルの動作理解:** カフェのWi-Fiやテザリング環境（NAT配下のプライベートIP環境）から自宅へ繋ぐ際、暗号化パケットがUDP 4500でカプセル化される挙動（CCNP頻出）をパケットレベルで確認できる。
* **家庭用回線の品質向上:** 日常使いの家族用通信は混雑に強いIPoE（transix）を通るため、夜間でも高速・低遅延が維持され、家族からクレームが出るリスクをゼロに抑えられる。追加費用なしで手持ちの契約アカウントをフル活用できる構成です。

---

# 3. 全体アーキテクチャとセグメント設計

## 3.1. 要件定義

* **完全な耐障害性分離:** 検証側のルータ群でOSPFの経路変更やフェイルオーバー試験を行っても、家庭用Wi-Fiネットワーク（192.168.24.0/24）には一切通信影響を与えない。
* **エンタープライズ3層階層型モデルの実装:**
  * **Core層:** NEC IX2106（WAN境界、PPPoE終端、グローバルIP直収、OSPFデフォルトルート配布）
  * **Distribution層:** Cisco C1111-8P（HSRP Active / IOS XE）および Cisco C891FJ（HSRP Standby / Classic IOS）
  * **Access層:** Cisco Catalyst 2960X × 2台（LACP EtherChannel ＋ Rapid-PVST+）
* **マルチベンダ動的ルーティング:** Core ⇔ Distribution間をOSPF Area 0で接続し、内部セグメントと対外経路を自動交換する。

## 3.2. IPアドレッシングとVLAN設計

| レイヤー / セグメント | ネットワークアドレス | VLAN ID | デフォルトGW | 主な収容機器・用途 |
| :--- | :--- | :---: | :--- | :--- |
| **家庭用生活網 (IPoE)** | `192.168.24.0/24` | - | `192.168.24.1` | 市販Wi-Fiルータ、家族の端末、家電（transix経由） |
| **検証 Transit (L3)** | `172.23.99.0/29` | - | なし (L3直結) | IX2106 (.1), C1111 (.2), C891FJ (.3) のOSPF交換用 |
| **検証 VLAN 2 (有線/サーバ)** | `172.23.1.0/24` | 2 | `172.23.1.254` (HSRP) | Proxmox 3ノード、DNS/DHCPミニPC、ラズパイ群、管理SVI |
| **検証 VLAN 10 (無線/検証端末)** | `172.23.10.0/24` | 10 | `172.23.10.254` (HSRP) | 検証用Wi-Fi AP、検証用クライアント端末 |

## 3.3. ネットワークトポロジ図

ONUの直下にギガビット対応のスイッチングハブを配置し、物理レイヤー（L1/L2）で家庭用（IPoE）と検証用（PPPoE）を分岐させています。

```text
                           [ インターネット ]
                                  │
                           [ NTT NGN網 ]
                                  │
                               [ ONU ]
                                  │
                 ┌────────────────┴────────────────┐
                 │     ONU直下 スイッチングハブ     │
                 └───────┬──────────────────┬──────┘
                         │                  │
        (EtherType: 0x86dd)                 │ (EtherType: 0x8863/0x8864)
         IPoEネイティブ (生活用)             │ PPPoEセッション (検証用)
                         │                  │
            ┌────────────┴────────┐ ┌───────┴────────┐
            │   家庭用Wi-Fiルータ  │ │  NEC IX2106    │ 【Core層 / WAN Gateway】
            │   (transix IPoE)    │ │  (PPPoE / IPsec)│ (グローバルIPv4直収)
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

# 4. 【最重要】ONU直下分岐における「IPv6プレフィックス奪い合い問題」と解決策

ONU直下にハブを置いて複数台のルータをぶら下げる構成（ONU直下分岐）を行う際、ネットワークスペシャリストやCCNPの座学知識だけでは見落としがちな物理網の罠が存在します。それが**「NTT NGN網におけるIPv6プレフィックスの排他仕様」**です。

## 4.1. なぜ家庭用IPoEルータが通信不能になるのか？

NTT東日本・西日本のNGN網（フレッツ光）では、回線の契約形態に応じて以下のいずれかの方式で宅内にIPv6プレフィックスを払い出します。

1. **ひかり電話契約あり（DHCPv6-PD方式）:** NGN網からルータに対して `/56` のプレフィックスが委譲（Prefix Delegation）される。
2. **ひかり電話契約なし（RA方式）:** NGN網からマルチキャスト（`ff02::1`）で `/64` のプレフィックスがルータのWANポートに通知（Router Advertisement）される。

ここで問題となるのが、NGN網のエッジ設備が持つ**「1契約回線につき、最初にIPv6要求（DHCPv6 SolicitやRSパケット）を出してきた1台の機器に対してのみプレフィックスをリースする」**という排他制御仕様です。

今回の構成では、家庭用Wi-FiルータがIPoE（transix）を喋るためにこのIPv6プレフィックスを絶対に取得しなければなりません。
しかし、もし検証用のルータ（IX2106）のWANポートで不用意にIPv6が有効化されていると、**IX2106が家庭用ルータよりも先にNGNのIPv6プレフィックスを奪い取ってしまう（またはプレフィックスの奪い合いが発生して競合する）**という事態が起きます。その結果、家庭用Wi-FiルータがIPv6アドレスを取得できず、transixのトンネルが張れなくなって生活用インターネットが全滅します。

## 4.2. イーサネットフレーム（EtherType）レベルでの完全解決

この問題を解決する設計は非常にシンプルかつ確実です。**「検証用のルータ（IX2106）のWAN物理ポート側で、IPv6のフレームを一切送受信させない（完全なIPv6盲目状態にする）」**ことです。

イーサネットフレームのレベルで見ると、PPPoEとIPoEは全く異なる言語（EtherType）を使用しています。

* **家庭用ルータ（IPoE）の通信:**
  * **EtherType `0x86dd`:** IPv6 ネイティブパケット。NGN網と直接通信してIPv6アドレスを取得し、VNEのAFTRと通信します。
* **検証用ルータ（PPPoE）の通信:**
  * **EtherType `0x8863`:** PPPoE Discovery（探索）
  * **EtherType `0x8864`:** PPPoE Session（PPPカプセル化データ通信）
  * PPPoEは網内のBAS（網終端装置）と1対1の仮想トンネルを張ってその内部でIPv4をやり取りするため、外部のネイティブIPv6通信とは物理線を共有していても完全に干渉しません。

```text
                  [ NTT NGN網 ]
                        │
                     [ ONU ]
                        │
             [ L2 スイッチングハブ ]
             ┌──────────┴──────────┐
             │                     │
 (EtherType: 0x86dd のみ)    (EtherType: 0x8863/0x8864)
             │                     │
   [ 家庭用IPoEルータ ]      [ 検証用Coreルータ (IX2106) ]
   ・NGNから見て「宅内の     ・WAN側物理ポートでIPv6を完全無効化
     IPv6端末はこれ1台だけ」  ・PPPoEトンネル通信のみを実行
   ・100%確実にIPv6を取得
```

IX2106のWAN側物理インターフェース（`GigaEthernet0.0`）において、IPv6スタックを起動せず、PPPoEクライアントのみをバインドさせます。これにより、IX2106はEtherType `0x86dd` のパケットを一切送信せず、NGNから流れてくるRAマルチキャストを受信しても即座にハードウェアで破棄します。

結果として、NGN側から見た宅内のIPv6要求元は**「家庭用Wi-Fiルータただ1台」**になり、アドレス競合や奪い合いが物理的・論理的に100%発生しなくなります。

---

# 5. Core-Distribution層のL3動的ルーティング設計（Cisco × NEC）

検証ネットワーク内部では、PPPoEを直収したNEC IX2106と、Distribution層である2台のCiscoルータ（C1111 / C891FJ）との間で**OSPF Area 0**を稼働させます。

マルチベンダでOSPFを組む場合、仕様解釈の違いからネイバーが上がらない、または非対称ルーティングが発生する代表的な2大トラップを対策します。

## 5.1. MTU不一致による「EXSTART / EXCHANGE」スタック対策

OSPFのネイバー確立プロセスでは、Down → Init → 2-Way を経て、DBD（Database Description）パケットを交換する **ExStart / Exchange** ステートへ遷移します。

この際、両端のインターフェースのMTUサイズが一致していないと、ルータはDBDパケットの受信を拒否し、ステートが `EXSTART` または `EXCHANGE` のまま停止します。

* **Ciscoの挙動:** デフォルトで自身のインターフェースMTU（通常1500）を厳密にチェックする。
* **NEC IXの挙動:** ファームウェアや設定によってMTUの解釈がズレることがある。

### 対策
対向するインターフェース（Transitセグメント: `172.23.99.0/29`）において、両者のIP MTUが確実に `1500` であることを設定・確認します。Cisco側で `ip ospf mtu-ignore` を打って無視させる回避策もありますが、LSA（リンクステート情報）が肥大化した際にパケットドロップの原因となるため、**実インターフェースのMTUを綺麗に揃えるのが設計の正道**です。

## 5.2. OSPFコスト計算基準帯域（Reference Bandwidth）の統一

OSPFのメトリック（Cost）計算式は以下の通りです。

$$\text{Cost} = \frac{\text{Reference Bandwidth (基準帯域)}}{\text{Interface Bandwidth (インターフェース帯域)}}$$

* **Cisco IOS / IOS XE のデフォルト:** **100 Mbps**
  * 100Mbpsのリンク: $\frac{100M}{100M} = \text{Cost } 1$
  * 1Gbps（1000Mbps）のリンク: $\frac{100M}{1000M} = 0.1 \rightarrow \text{Cost } 1$（切り上げ）
  * 100Mbpsも1Gbpsもすべてコスト「1」になり、ギガビット環境で帯域差を正しく評価できない。
* **NEC UNIVERGE IX のデフォルト:** 機種や世代により **1 Gbps** または 10 Gbps

基準帯域が異なると、Cisco側とNEC側で計算されるルートのメトリックがチグハグになり、行きと帰りで経路が異なる「非対称ルーティング」が発生します。Cisco側で必ず以下を投入し、ネットワーク全体の基準帯域を統一します。

```text
router ospf 1
 auto-cost reference-bandwidth 1000
```

## 5.3. デフォルトルート配信とパッシブ設定

* **Coreルータ（IX2106）:** インターネット向けPPPoEセッションを持っているため、OSPF内で `default-information originate always` を設定し、配下のCiscoルータ群へ外向きデフォルトルート（`O*E2 0.0.0.0/0`）を一括配信します。
* **Distributionルータ（C1111 / C891FJ）:** 内部セグメント（VLAN 2: `172.23.1.0/24`、VLAN 10: `172.23.10.0/24`）をOSPFに広告し、外から戻ってきたパケットが正確に下位へ届くようにします。
* **Passive-Interfaceの徹底:** サーバやPCが接続される内部LANインターフェース（SVI Vlan2, Vlan10）に対して、不要なOSPFパケットを送出しないよう `passive-interface` を設定します。

---

# 6. Distribution層におけるデフォルトゲートウェイ冗長化（HSRP）

配下のサーバ群（Proxmox、DNS、Zabbix等）やWi-Fiクライアントにとって、デフォルトゲートウェイの停止は完全な通信断を意味します。これを防ぐため、2台のCiscoルータ（C1111とC891FJ）間で **HSRP（Hot Standby Router Protocol）バージョン2** を稼働させます。

## 6.1. HSRPv2設計パラメータ

* **仮想IPアドレス（VIP）:**
  * VLAN 2: `172.23.1.254`
  * VLAN 10: `172.23.10.254`
* **優先度（Priority）と役割分担:**
  * **Cisco C1111-8P:** `Priority 110` ＋ `Preempt` 有効 → **【Activeルータ】**
  * **Cisco C891FJ:** `Priority 100` (デフォルト) → **【Standbyルータ】**

高性能なC1111（IOS XE）を通常時の通信担当（Active）とし、C1111の障害時やメンテナンス時のみ、C891FJ（Classic IOS）が自動的に仮想IPと仮想MACアドレスを引き継ぎます。`preempt` を有効にしているため、C1111の復旧後は自動的にActiveへ切り戻されます。

## 6.2. Accessスイッチ（Catalyst 2960X）とのRapid-PVST+連携

Distributionルータ2台とAccessスイッチ2台（Catalyst 2960X #1, #2）は、耐障害性を極限まで高めるために**クロス接続（L2メッシュトポロジ）**で配線されています。

ここで重要になるのが、**「STPルートブリッジ」と「HSRP Activeルータ」の位置の同期**です（CCNP ENCOR設計分野の鉄板項目）。

* **不整合のアンチパターン:**
  もしSTPルートブリッジを「2960X #2」にし、HSRP Activeを「C1111（2960X #1直結）」にしてしまうと、端末から出たフレームが「スイッチ間リンクを渡ってからルータへ向かう」という無駄な迂回（トロンボーン現象）が発生します。
* **最適化設計:**
  C1111が直結されている **Catalyst 2960X #1 をSTP Primary Root Bridge（Priority 4096）** に固定し、最短経路でActiveルータへパケットが抜けるようにコストを最適化します。

---

# 7. 実機完全コンフィグ集

本環境で稼働している各機器の設定投入用コンフィグです。
Cisco機器の設定保存コマンドは、実務標準の **`copy run start`** に統一しています。

## 7.1. 検証用Coreルータ：NEC UNIVERGE IX2106（PPPoE / NAT / OSPF）

WANポート（GE0）でPPPoE接続してグローバルIPv4アドレスを取得・NAPTし、下位（Transit）へOSPFでデフォルトルートを配布する基本構成です。

```text
enable-config

hostname IX2106

! --- WAN側物理ポート（GE0: IPv6を起動させずPPPoEのみバインド） ---
interface GigaEthernet0.0
  no ip address
  no shutdown
exit

! --- PPPoEクライアント設定 ---
interface pppoe1
  ip address negotiated
  pppoe user <PPPOE_USERNAME>
  pppoe password <PPPOE_PASSWORD>
  pppoe server-name ""
  ip tcp adjust-mss 1414
  ip napt enable
  no shutdown
exit

! --- Transit LANポート（Cisco対向ルーテッドポート） ---
interface GigaEthernet1.0
  ip address 172.23.99.1/29
  ip tcp adjust-mss 1414
  no shutdown
exit

! --- NAPT（IPマスカレード）設定 ---
ip ufs-cache enable
ip napt translation max 20000
ip napt translation auto-interface-address pppoe1

! --- デフォルトルート（外向きPPPoEへ向ける） ---
ip route default pppoe1

! --- OSPF Area 0 設定 ---
router ospf 1
  router-id 172.23.99.1
  network 172.23.99.0/29 area 0.0.0.0
  ! ディストリビューションルータ群へデフォルトルートを広告
  default-information originate always
exit

! --- 設定保存 ---
exit
write memory
```

## 7.2. 検証用Distributionルータ #1：Cisco C1111-8P（OSPF / HSRP Active）

Transit接続、内部VLANの集約、およびHSRP Activeとして動作する主軸ルータです。

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
 ! Transitセグメント
 network 172.23.99.0 0.0.0.7 area 0
 ! 内部セグメント広報
 network 172.23.1.0 0.0.0.255 area 0
 network 172.23.10.0 0.0.0.255 area 0
 ! 内部LANへHelloパケットを送出しない
 passive-interface Vlan2
 passive-interface Vlan10
exit

! --- 設定保存 ---
end
copy run start
```

## 7.3. 検証用Distributionルータ #2：Cisco C891FJ（OSPF / HSRP Standby）

C1111のスタンバイ機として待機し、障害発生時に無瞬断でゲートウェイを引き継ぐルータです。

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

! --- 設定保存 ---
end
copy run start
```

## 7.4. Accessスイッチ：Cisco Catalyst 2960X（LACP / RSTP）

スイッチ間をLACPで束ね、Rapid-PVST+でループを制御しつつエッジポートを保護します。

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

! --- 設定保存 ---
end
copy run start
```

---

# 8. ダウンタイム数秒で完結する安全な切り替え手順（SOP）

本番稼働中の環境でネットワーク機器を入れ替える際、最もやってはならないのが「ぶっつけ本番で配線を挿し替え、コンソールで手動設定を打ち込むこと」です。トラブルが発生した瞬間に何が原因か切り分けられなくなります。

今回は以下の**「5段階の段階的移行（フェーズアプローチ）」**を策定して実行しました。

## Phase 1: 机上事前設定（完全オフライン）
新設する機器（IX2106、C1111）を検証デスクに並べ、PCと1対1でコンソール接続して、上記の設定をすべて投入・保存（`copy run start` / `write memory`）しておきます。この段階では本番のLANケーブルは1本も接続しません。

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
ONUからのLANケーブルを5ポートハブに挿し、家庭用ルータ（IPoE）とIX2106（PPPoE）へそれぞれ配線します。これで家庭用（IPoE）と検証用（PPPoE）の完全二重化が完了します。

---

# 9. 正常性確認コマンド集と期待される出力

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

### ④ PPPoEグローバルIP取得の確認（NEC IX2106側）
```bash
Router# show ip interface pppoe1

Interface pppoe1 is up, line protocol is up
  IP address: 203.0.113.15/32 (negotiated)
```
* **確認ポイント:** プライベートIPではなく、プロバイダから払い出された本物のグローバルIPv4アドレスが割り当たっていること。

---

# 10. おわりに＆今後の展望

本構成の実現により、**「どれだけ検証環境で大規模な設定変更や障害試験を行っても、家庭の生活用ネットワークには1ミリ秒の影響も与えない」**という究極のサンドボックス環境を手に入れることができました。
さらに、検証用ルータ（IX2106）にグローバルIPv4アドレスを直収できたことで、外出先のPCからの**直接着信IPsec-VPNリモートアクセスの検証**が可能になりました。

ネットワークの強固な足回りが完成したため、今後はGitHubリポジトリを活用した**「NetDevOps（GitOps）」**のフェーズへ進める予定です。

1. **Docs as Codeの推進:** IPアドレス管理（IPAM）やトポロジ図をMarkdownやdraw.ioでコード管理し、Gitで差分追跡を行う。
2. **自動化パイプラインの構築:** Cisco C1111（IOS XE）のRESTCONF/NETCONFを活用し、PythonスクリプトやAnsibleによるコンフィグの自動バックアップと整合性チェックを実装する。
3. **統合監視と認証統合:** Proxmoxクラスタ上にオープンソースSIEMである「Wazuh」をデプロイして全機器のSyslogを集約し、さらに「FreeIPA」を立ち上げてLinuxサーバ群の鍵管理・集中認証基盤を構築する。

物理環境特有の泥臭いトラブルシューティング（EtherTypeやMTU、プロトコルの癖）を乗り越えた経験は、そのまま現場での設計力や障害対応力に直結します。
自宅ラックやホームラボの構築に挑んでいるエンジニアの方々に、本記事が少しでも参考になれば幸いです！