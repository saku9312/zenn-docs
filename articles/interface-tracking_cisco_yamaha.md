---
title: "YAMAHAとCiscoでVRRPインターフェーストラッキングの設定"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# YAMAHAとCiscoでインターフェーストラッキングの設定
別途記事にするが、VRRPでCisco 891fjとYAMAHA RTX810の2台で設定を行っており、外部との疎通でシャットダウン＆切り替えをさせたいと思い設定を試してみた。

## YAMAHA RTX810　icmpでインターフェイストラッキング
   
    #60秒で8.8.8.8宛にPing3回
    ip keepalive 1 icmp-echo 60 3 8.8.8.8
    
    #Keepaliveがyesの場合static route追加
    ip route default gateway 10.10.1.1 keepalive 1
    
    #ルートが残っている場合にルーター使用OKと判断
    ip lan1 vrrp shutdown trigger 2 route default 10.10.1.1

show status ip keepalive　で状態確認

## Cisco 891fj icmpでインターフェイストラッキング

    #ip sla でicmp設定（Gi8指定だと常にDOWN）
    ip sla 1
        icmp-echo 8.8.8.8 source-interface Dialer 1
        frequency 10
    ip sla schedule 1 life forever start-time now

    #track設定
    track 1 ip sla 1 reachability
        delay down 31　→ちょっとラグを持たせる
    #一応ポートダウン時の判定も入れておく
    track 2 interface Gigaethernet 8 line-protocol

    #vrrpに適用
    interface vlan 2
        vrrp 2 track 1
        vrrp 2 track 2

#状態確認はtrack全体とslaで確認
show track
show ip sla statistices

# 結果
Ciscoがアクティブだった場合にONU-HGW間のケーブルを抜くと、自動的にYAMAHAへ切り替わり、Ciscoが復旧したときにCiscoがアクティブに戻ることを確認できた。
別途、プリエンプトの記事も作成したい。