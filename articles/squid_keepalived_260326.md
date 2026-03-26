---
title: "冗長化したプロキシサーバー構築の検証ログ（SquidとKeepalived）"
emoji: "📑"
type: "tech"
topics: ["Squid", "keepalived", "プロキシサーバー"]
published: true
---
# はじめに
Squidの導入案件がいくつかあるのを見かけていて、自分も触っておいた方がよいと思って検証をしてみました。
せっかくなので、冗長性の担保とブラックリストでのフィルタリングを行ってみます。

Keepalivedで複数のプロキシサーバーで仮想IPを持つようにして、お互いPingで疎通を確認してどちらかだけが仮想IPを持つように設定してみました。
初めて触りましたが、マルチキャストの疎通確認ができなかったりしましたが、何とかイメージしていた形までは持っていけました。

# Squidの設定
## 1. Squidインストールコマンド
以下のコマンドを入力し、Squidをインストールします。

```
dnf install -y squid
```

次に、ファイアウォール設定を変更し、3128/tcpを開放します。
```
firewall-cmd --add-port=3128/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

Squidのコンフィグを編集し、ブロックサイト設定の部分を追記します。
```
vi /etc/squid/squid.conf

    acl blocked_sites dstdomain .youtube.com
    http_access deny blocked_sites
```

サービスを起動し、自動起動設定を行います。ステータスの確認も行っておきましょう。
```
systemctl start squid
systemctl enable squid
systemctl status squid
```

## 2. プロキシサーバーの登録
クライアントPCの設定から、プロキシサーバーを登録します。
IPアドレス：3128
![](https://storage.googleapis.com/zenn-user-upload/1b5b4ea0814e-20260326.png)

上記の設定の場合、youtubeへアクセスした際にはブロックされます。（http、https問わず）
![](https://storage.googleapis.com/zenn-user-upload/5b41676705c6-20260326.png)

 上記の設定の場合、youtubeへアクセスした際にはブロックされます。（http、https問わず）
 
# フィルタリングするドメインリストを別で作成する
## 1.	ブラックリスト用のファイル作成
ブラックリストを別ファイルで管理した方が効率が良いため、ブラックリストファイルを作成します。
```
mkdir /etc/squid/acl
vi /etc/squid/acl/blocked_domains.txt
# ブラックリストドメイン
    .facebook.com
    .youtube.com
    .yahoo.co.jp
chown -R squid:squid /etc/squid/acl/*
```

ACLに定義します。注意点としては、上から順にブロックされるため、acl localnetより上に入れます。（ホワイトリストならdeny all より上）
```
vi /etc/squid/squid.conf
    acl blocked_domains dstdomain "/etc/squid/acl/blocked_domains.txt"
```

サービスを再起動して、構文チェックをしておきます。
```
systemctl restart squid
squid -k parse
```

構文に問題がなければ、プロキシサーバーを経由したインターネットアクセスを試します。ブロックリストファイルに記載したドメインは停止し、それ以外は通過します。例ではGoogleを通しますが、Yahooをブロックしています。 
![](https://storage.googleapis.com/zenn-user-upload/bf5ed1f1373f-20260326.png)

![](https://storage.googleapis.com/zenn-user-upload/092bb9e8c84d-20260326.png)

# SSL復号と通信内容傍受の実施（8080/tcpに変更）
## 1.	SELinuxの無効化
SELinuxが有効になっている場合、証明書を参照することができないため、無効にしておきます。
```
vi /etc/selinux/config
# enforced →　disabledに変更
systemctl reboot
getenforce
```

今度はポートを変更するため、ファイアウォール設定も追加しておきます。
```
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

## 2.	SSL証明書の作成（自己証明書）
追加でnet-toolsをインストールします。
```
dnf install net-tools
```

プロキシサーバーを使用する際に動的に中間証明書が生成されますが、その署名に使用する自己証明書を作成します。
```
mkdir -p /etc/squid/ssl_cert
cd /etc/squid/ssl_cert
openssl dhparam -outform PEM -out dhparam.pem 2048
openssl req -new -newkey rsa:2048 -nodes -x509 -keyout myCA.key -days 3650 -out myCA.pem
    Country Name (2 letter code) [XX]:JP　←★JPを指定
    State or Province Name (full name) []:Aichi　←★任意で指定
    Locality Name (eg, city) [Default City]:　←★そのままエンター
    Organization Name (eg, company) [Default Company Ltd]:　←★そのままエンター
    Organizational Unit Name (eg, section) []:　←★そのままエンター
    Common Name (eg, your name or your server's hostname) []:Squid-cert　←★任意
    Email Address []:　←★そのままエンター
```

証明書のハッシュ値を確認し、一致することを確認しておきます。
```
openssl x509 -in /etc/squid/ssl_cert/myCA.pem -noout -modulus | openssl md5
openssl rsa -in /etc/squid/ssl_cert/myCA.key -noout -modulus | openssl md5
```

## 3.	証明書DBの作成
Squidは動的証明書を作成するためにsslcrtdを使用し、そのために証明書DBを作成する必要があります。
```
rm -rf /var/lib/squid/ssl_db
/usr/lib64/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 64MB
chown -R squid:squid /var/lib/squid/ssl_db
```

## 4.	squid.confの編集
証明書とSSLの復号に関する設定を追記します。
```
sslcrtd_program /usr/lib64/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 64MB
sslproxy_cert_error allow all
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all
```

続いて、ポートの設定と証明書の配置場所を入力します。
```
http_port 8080 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=64MB tls-cert=/etc/squid/ssl_cert/myCA.pem tls-key=/etc/squid/ssl_cert/myCA.key tls-dh=prime256v1:/etc/squid/ssl_cert/dhparam.pem
```

サービスを再起動して、設定を反映させます。
```
systemctl restart squid
systemctl status squid
```

## 5.	クライアントへの証明書の登録
サーバー側で作成した、myCA.pemをクライアント端末へインストールします。
Windowsの場合、「ローカルコンピューター＞信頼されたルート証明機関＞証明書」 にインストールします。
![](https://storage.googleapis.com/zenn-user-upload/2f9d34c344ba-20260326.png)

## 6.	アクセスログの確認
ログを確認すると、今まではCONNECTメソッドしか見えなかったのが、GETやPOSTのログまで表示されるようになります。
TCP_MISSは、キャッシュになかったことで外部サイトへ取りにいったためで、エラーではありません。むしろキャッシュ（TCP_HIT）できないサイトが多いため、特に問題はありません。
![](https://storage.googleapis.com/zenn-user-upload/fdb6c3dc5a42-20260326.png)

## 7.	補足：squid.conf全文
```
# Recommended minimum configuration:
#
 
# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 0.0.0.1-0.255.255.255  # RFC 1122 "this" network (LAN)
acl localnet src 10.0.0.0/8             # RFC 1918 local private network (LAN)
acl localnet src 100.64.0.0/10          # RFC 6598 shared address space (CGN)
acl localnet src 169.254.0.0/16         # RFC 3927 link-local (directly plugged) machines
acl localnet src 172.16.0.0/12          # RFC 1918 local private network (LAN)
acl localnet src 192.168.0.0/16         # RFC 1918 local private network (LAN)
acl localnet src fc00::/7               # RFC 4193 local private network range
acl localnet src fe80::/10              # RFC 4291 link-local (directly plugged) machines
 
acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280         # http-mgmt
acl Safe_ports port 488         # gss-http
acl Safe_ports port 591         # filemaker
acl Safe_ports port 777         # multiling http
acl CONNECT method CONNECT
 
acl blocked_sites dstdomain "/etc/squid/acl/blocked_domains.txt"
 
# Cert Setting
sslcrtd_program /usr/lib64/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 64MB
sslproxy_cert_error allow all
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all
 
#
# Recommended minimum Access Permission configuration:
#
# Deny requests to certain unsafe ports
http_access deny !Safe_ports
 
# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports
 
# Only allow cachemgr access from localhost
http_access allow localhost manager
http_access deny manager
 
http_access deny blocked_sites
 
# We strongly recommend the following be uncommented to protect innocent
# web applications running on the proxy server who think the only
# one who can access services on "localhost" is a local user
#http_access deny to_localhost
 
#
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#
 
# Example rule allowing access from your local networks.
# Adapt localnet in the ACL section to list your (internal) IP networks
# from where browsing should be allowed
http_access allow localnet
http_access allow localhost
 
# And finally deny all other access to this proxy
http_access deny all
 
# Squid normally listens to port 3128
#http_port 3128
#http_port 8080
 
# port setting 8080
http_port 8080 ssl-bump cert=/etc/squid/ssl_cert/myCA.pem key=/etc/squid/ssl_cert/myCA.key generate-host-certificates=on dynamic_cert_mem_cache_size=64MB tls-dh=prime256v1:/etc/squid/ssl_cert/dhparam.pem
 
# Uncomment and adjust the following to add a disk cache directory.
#cache_dir ufs /var/spool/squid 100 16 256
 
# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid
 
#
# Add any of your own refresh_pattern entries above these.
#
refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .               0       20%     4320
```

# プロキシサーバーの冗長化（keepalived使用）
## 1.	2台目の構築 
冗長化するために、同じ設定のSquidをもう一台構築します。（ファイアウォールの開放を忘れず）
2台目にも、1台目と同じ証明書（myCA.key、myCA.pem）をコピーしておきます。
```
mkdir -p /etc/squid/acl
mkdir -p /etc/squid/ssl_cert
scp /etc/squid/ssl_cert/myCA.key root@IPアドレス:/etc/squid/ssl_cert/myCA.key
scp /etc/squid/ssl_cert/myCA.pem root@IPアドレス:/etc/squid/ssl_cert/myCA.pem
scp /etc/squid/ssl_cert/dhparam.pem root@IPアドレス:/etc/squid/ssl_cert/dhparam.pem
```

1から構築するのが面倒なので、configとブロックリストもコピーします。
```
scp /etc/squid/ acl/blocked_domains.txt root@IPアドレス:/etc/squid/acl/blocked_domains.txt
scp /etc/squid/squid.conf root@IPアドレス:/etc/squid/squid.conf
```

Squidは動的証明書を作成するためにsslcrtdを使用し、そのために証明書DBを作成する必要があります。
```
rm -rf /var/lib/squid/ssl_db
mkdir /var/lib/squid/
/usr/lib64/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 64MB
chown -R squid:squid /var/lib/squid/ssl_db
```

2台目のSquidでも証明書でGETやPOSTが見れることを確認しておきます。

## 2.	keepalivedのインストール
Squidを冗長化する場合は、1台目と同じ設定のSquidをもう1つ用意し、keepalivedをインストールして仮想IPに対してアクセスをさせるように設定します。
```
dnf install keepalived
```

両方のサーバーに、VIPの共有設定を追加します。
```
echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee -a /etc/sysctl.conf
sysctl -p
```

## 3.	keepalived.confの設定
1台目に以下の設定を追加します。
```
# /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0                 # NIC名を実環境に合わせる
    virtual_router_id 51
    priority 120
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass s3cr3t
    }
    virtual_ipaddress {
        192.168.9.150/24 dev eth0  # VIP
    }

    # Squid死活と連動させる例（任意）
    track_script {
        chk_squid
    }
}

vrrp_script chk_squid {
    script "/usr/bin/pgrep squid"  # 0:OK/非0:NG
    interval 5
    weight -20                     # 失敗時priority低下→VIP移動
}
```

2台目に以下の設定を追加します。
```
# /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass s3cr3t
    }
    virtual_ipaddress {
        192.168.0.100/24 dev eth0
    }

    track_script {
        chk_squid
    }
}

vrrp_script chk_squid {
    script "/usr/bin/pgrep squid"
    interval 5
    weight -20
}
```

## 4.	マルチキャストパケットの疎通設定
VRRPの死活監視のためにマルチキャストパケットを送受信する必要があるため、以下の設定を追加します。
```
firewall-cmd --direct --add-rule ipv4 filter INPUT 1 -i eth0 -d 224.0.0.18 -p vrrp -j ACCEPT
firewall-cmd --direct --add-rule ipv4 filter OUTPUT 1 -o eth0 -d 224.0.0.18 -p vrrp -j ACCEPT
firewall-cmd --runtime-to-permanent
firewall-cmd --direct --get-all-rules
ipv4 filter OUTPUT 1 -o eth0 -d 224.0.0.18 -p vrrp -j ACCEPT
ipv4 filter INPUT 1 -i eth0 -d 224.0.0.18 -p vrrp -j ACCEPT
```

VRRPの間でマルチキャストがやり取りされているか確認します。
```
tcpdump -i eth0 proto 112
```

上記でうまくいかず、両方のサーバーで仮想IPを持ってしまう場合は、直接それぞれのサーバーが持つIPを指定することができます。
```
vi /etc/keepalived/keepalived.conf
    virtual_ipaddressの下に追加（自分のIPも入れる）
    unicast_peer {
        192.168.9.103
    192.168.9.118
    }
```

サービスを再起動して、仮想IPが片方だけに割り当てられていることを確認します。
```
systemctl restart keepalived
ip a
```

## 5.	Squidの動作確認
仮想IPが片方だけに割り当てられていることを確認したら、ブラウザからアクセスできることを確認します。
![](https://storage.googleapis.com/zenn-user-upload/f4cd7ad0633c-20260326.png)
 
6.	障害時の切り替わりを確認する
クライアントへ仮想IPでプロキシサーバーを登録します。
![](https://storage.googleapis.com/zenn-user-upload/1dc97fba5069-20260326.png)

MASTER機が動作している状態でブラウザからアクセスすると、問題なくWebページが表示されます。
![](https://storage.googleapis.com/zenn-user-upload/c9cd5790d1a2-20260326.png) 

![](https://storage.googleapis.com/zenn-user-upload/08d1d56a9342-20260326.png)
 
MASTER機をシャットダウンすると、自動的にBACKUP機へIPが割り当てられます。そちらでも問題なくWebページが確認できれば成功です。証明書は同じものを使用できるので、ユーザーは意識する必要がありません。
マスター機
![](https://storage.googleapis.com/zenn-user-upload/dc54f3d09296-20260326.png)

バックアップ機
![](https://storage.googleapis.com/zenn-user-upload/291789b4b5c5-20260326.png)

![](https://storage.googleapis.com/zenn-user-upload/6a1ee68f3a28-20260326.png)

# おわりに
Squidの設定はもっと深いものが多いと思いますし、この内容はあくまでフィルタリングが動作するかどうかに視点を置いています。
内容の浅さは仕方ないとして、今回やりたかったSquidの基本設定と冗長化は無事に実現できました。

Keepalivedでは3台以上の冗長化も可能なため、複数のサーバー障害に対応できる形も構成できるようになります。
プロキシサーバーだけでなく、DHCPやDNSの冗長化もこのやり方ならできるかなと。ロードバランスではないですが。。。
今度はそういった負荷分散も検証しておこうと思います。