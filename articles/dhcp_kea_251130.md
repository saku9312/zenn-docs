---
title: "DHCPサーバー（kea）で複数セグメントへのIP配布環境を構築"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに
Windows Serverの評価版でDHCPサーバーを立てていたが、有効期限が近付いてきたためにRedHatで構築してみる。

## 構成
- Red Hat Linux ENterprise 10.0
- kea
- ネットワークはボンディングし、「bond0」を使用

# DHCPサーバー構築
## kea のインストール
DHCPサーバーとして、「kea」をインストールする。
```
$ sudo dnf install kea
```

## DHCPサーバーの設定
1. /etc/kea/kea-dhcp4.conf をバックアップしておく。
```
$ cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.org
```
2. /etc/kea/kea-dhcp4.conf の設定を変更する。今回は2つのサブネット（172.23.1.0/24,172.23.10.0/24）の端末にIPを配布できるように設定する。
```
$ sudo vi /etc/kea/kea-dhcp4.conf
```
```
{
    "Dhcp4": {
        "interfaces-config": {
            "interfaces": [ "bond0/172.23.1.XXX" ],
            "dhcp-socket-type": "raw"
        },

        "subnet4": [
            {
                "id": 1,
                "subnet": "172.23.1.0/24",
                "pools": [ { "pool": "172.23.1.151 - 172.23.1.200" } ],
                "option-data": [
                    {
                        "name": "routers",
                        "data": "172.23.1.254"
                    },
                    {
                        "name": "domain-name-servers",
                        "data": "172.23.1.10, 8.8.8.8"
                    }
                ] 
            }, 
            {
                "id": 2,
                "subnet": "172.23.10.0/24",
                "pools": [ { "pool": "172.23.10.151 - 172.23.10.200" } ],
                "option-data": [
                    {
                        "name": "routers",
                        "data": "172.23.10.254"
                    },
                    {
                        "name": "domain-name-servers",
                        "data": "172.23.1.10, 8.8.8.8"
                    }
                ]
            }
        ]
    }
}

```

## kea の起動設定
1. DHCPサーバーの起動、自動起動を設定する。
```
$ sudo systemctl start kea-dhcp4
$ sudo systemctl enable kea-dhcp4
```

## ファイアウォールの設定
1. 53/udpを開放しておく。
```
$ sudo firewall-cmd --add-port=67/udp --permanent
$ sudo firewall-cmd --add-port=68/udp --permanent
```
2. firewalld を読み込み直し、DHCPサーバーへ到達できるようにする。
```
$ sudo firewall-cmd --reload
```


## DHCPからのIP取得
今回はWindowsOSの端末から、IPが配布されているか確認する。
1. コマンドプロンプトやPowerShellを起動する。
2. 以下のコマンドで自動取得したIPを解除する。
```
ipconfig /release
```
3. 以下のコマンドで、新しいDHCPサーバーからIPを取得する。
```
ipconfig /renew
```
4. 以下のコマンドで、IPやDNSなどが取得できたか確認する。
```
ipconfig /all
```
```
   接続固有の DNS サフィックス . . . . .:
   説明. . . . . . . . . . . . . . . . .: XXXXXXXXXXXXXXXXXXXXXXXX
   物理アドレス. . . . . . . . . . . . .: XX-XX-XX-XX-XX-XX
   DHCP 有効 . . . . . . . . . . . . . .: はい
   自動構成有効. . . . . . . . . . . . .: はい
   IPv4 アドレス . . . . . . . . . . . .: 172.23.10.151(優先)
   サブネット マスク . . . . . . . . . .: 255.255.255.0
   リース取得. . . . . . . . . . . . . .: XXXX年XX月XX日 XX:XX:XX
   リースの有効期限. . . . . . . . . . .: XXXX年XX月XX日 XX:XX:XX
   デフォルト ゲートウェイ . . . . . . .: 172.23.10.254
   DHCP サーバー . . . . . . . . . . . .: 172.23.1.10
   DNS サーバー. . . . . . . . . . . . .: 172.23.1.10
                                          8.8.8.8
   NetBIOS over TCP/IP . . . . . . . . .: 有効
```

# まとめ
DHCPサーバーの構築を試してみたが、configがJSON形式なのが少し戸惑った。
ネット上に情報はあるが、実際に2つのセグメントへIPを割り振る記事が少なそうだったため、記録として残しておく。
WindowsServerならDHCPの冗長化ができるが、これだと割り当てるIP帯を分けて冗長化するしかない？ため、WindowsServerの方が楽ではありそう。

参考資料：https://wiki.archlinux.jp/index.php/Kea
