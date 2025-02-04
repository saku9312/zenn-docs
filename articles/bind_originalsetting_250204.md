---
title: "BINDの設定をしてDNS解決させてみた"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
Ubuntu24.04にBINDを入れて、LAN内のDNS解決ができるように設定してみる。
インストールなどは簡単なので、後は必要があればFWの穴あけなどはやっておこう（UDP53）

## やってみた
Windows側からも名前解決できている。
https://changineer.info/vmware/hypervisor/vmware_ubuntu_dns.html

    #bindインストール
    apt install bind9 bind9utils

    # confファイルを編集
    vi /etc/bind/named.conf
        include "/etc/bind/named.conf.local";
        // BIND設定オプションファイル
        include "/etc/bind/named.conf.options";
        // BINDゾーン情報ファイル
        include "/etc/bind/named.conf.default-zones";

    # Ubuntuはnamed.conf.localで設定
    sudo vi /etc/bind/named.conf.local
        zone "saku.XXX" {
            type master;
            file "/etc/bind/saku.XXX" ;
        };
        zone "1.23.172.in-addr.arpa" {
            type master;
            file "/etc/bind/1.23.172.db" ;
        };

    # オプションの設定
    vi /etc/bind/named.conf.options
    // SUBNET
        acl internal-network {
            172.23.1.0/24;
            172.23.10.0/24;
        };

        options {
            directory "/var/cache/bind";
        forwarders {
                8.8.8.8; // Google Public DNS
                8.8.4.4; // Google Public DNS Secoundary
        };
        // ACLの指定
        allow-query {
                localhost;
                internal-network;
        };

        recursion yes;
        forward only;
        dnssec-validation auto;
        // いったんipv6は許可しない
        listen-on-v6 { none; };
        // バージョンは教えない
        version "I won't tell you my version.";
     };

    # 正引きファイル
    vi /etc/bind/saku.XXX
        $TTL 86400
        @       IN      SOA     ns.saku.XXX. root.saku.XXX. (
                        2024120501      ;Serial
                        3600            ;Refresh
                        1000            ;Retry
                        604800          ;Expire
                        86400           ;Minimum TTL
        );
        @       IN NS   ns.saku.com.
        @       IN A    172.23.1.1      ;DNSサーバーのIP
        ns      IN A    172.23.1.1
        saku-samba      IN CNAME        saku-Ubuntu1
        saku-Ubuntu1    IN A    172.23.1.1
        saku-zabbix     IN CNAME        saku-Ubuntu2
        saku-Ubuntu2    IN A    172.23.1.2

    # 逆引きファイル
    vi /etc/bind/1.23.172.db
        $TTL 86400
        @       IN      SOA     ns.saku.XXX.    root.saku.XXX.  (
                    2024120501      ;Serial
                    3600            ;Refresh
                    1000            ;Retry
                    604800          ;Expire
                    86400           ;Minimum TTL
    
        );
        @       IN NS   ns.saku.XXX.
        1       IN PTR  ns.saku.XXX.
        2       IN PTR  saku-Ubuntu2.saku.XXX.

    # UFWの許可
    ufw allow 53/tcp
    ufw allow 53/udp

    # 設定の反映
    systemctl restart bind9 bind9utls
    systemctl status bind9

    # 通信確認
    dig saku.XXX @127.0.0.1
    dig -x 172.23.1.1 @127.0.0.1

    # ローカル側のDNS設定
    vi /etc/netplan/99-cloud-init.yaml →nameserver変更
    もしくは
    vi /etc/resolv.conf →DNSserverのIPに変更

## CNAMEで別名を入れたらエラーが発生
CNAMEで別名を作成したら

    sudo:unable to resolve host saku-Ubuntu1: Name or service not known

と表示されて実行できなくなった。
どうやらホスト名が解決できなかったことが問題らしい。

    vi /etc/hosts
        127.0.1.1 hostname.domainname saku-Ubuntu1

と変更したところ実行できた。
ローカル側のホスト名として、固定するものはこちらに記載が必要。
必要ならLAN側のIPとの紐づけもここに入れておこう。