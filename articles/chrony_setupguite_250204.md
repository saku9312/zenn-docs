---
title: "LinuxでのNTPサーバー（chrony）構築"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
自宅環境での時刻設定を同期させるために、LinuxでNTPサーバーを立てて、すべての機器がそこを参照できるように設定したくなった。
Linuxは昔はntpd、現在はchronyを使えばいいとのことで、比較的簡単に設置できた。

## NTP(chrony)のセットアップ
メイン：ntp.nict.jp
バックアップ：0.jp.pool.ntp.org / 1.jp.pool.ntp.org / 2.jp.pool.ntp.org
jp.pool.ntp.orgは、NTP POOL PROJECTのNTPサーバのこと。
https://tech.willserver.asia/2022/12/23/lix-ubt2204-ntp-server/



    # インストール
    apt install chrony

    # 有効化
    systemctl enable chrony
    systemctl start chrony

    # confのバックアップ
    cp -a /etc/chrony/chrony.conf /etc/chrony/chrony.conf.org

    # confの編集、pool部分はコメントアウト
    vi /etc/chrony/chrony.conf
        # pool ntp.ubuntu.com        iburst maxsources 4
        # pool 0.ubuntu.pool.ntp.org iburst maxsources 1]
        # pool 1.ubuntu.pool.ntp.org iburst maxsources 1
        # pool 2.ubuntu.pool.ntp.org iburst maxsources 2
        # minpollとmaxpollを指定するのがWANのNTPを指定する際のマナーらしい
        # ntpサーバーはメイン1台、バックアップ3台設定しておく
        server ntp.nict.jp minpoll 6 maxpoll 8
        server 0.jp.pool.ntp.org minpoll 6 maxpoll 8
        server 1.jp.pool.ntp.org minpoll 6 maxpoll 8
        server 2.jp.pool.ntp.org minpoll 6 maxpoll 8

        # 最終行に許可IPを設定
        # Allow IPs
            deny all
            allow 172.23.1.0/24
            allow 172.23.10.0/24

    # サービスの再起動
    systemctl restart chrony

    # NTPサーバーとの通信確認
    chronycc sources

## 補足事項
<server / poll> <NTPサーバのホスト名> と記載することで、上位のNTPサーバを設定できます。
オプションで記載しているminpollとmaxpollは上位のNTPサーバに問い合わせを行う間隔です。
2の階乗秒の間隔で取得しに行きます。(minpoll 6であれば最小64秒間隔、maxpoll 10であれば最大1024秒間隔)
これについては、内部のNTPサーバを参照するのであれば特に気にする必要はありませんが、外部のNTPサーバに問い合わせを行う場合、マナーとして設定しておきましょう。

## NTPクライアント側の設定
これでクライアントとしての設定も反映させる。

    # chronyのインストール
    apt install chrony

    # chronyのコンフィグのバックアップ&編集
    cp -a /etc/chrony/chrony.conf /etc/chrony/chrony.conf.org
    vi /etc/chrony/chrony.conf

    # 参照元を指定（poolの入っているものはコメントアウトして使わない）
    pool 172.23.1.2 iburst　を追加

    # サービスの再起動と有効化
    systemctl enable chrony
    systemctl restart chrony
    systemctl status chrony
    chronyc sources

    # Cisco側の設定
    ntp server 172.23.1.2
    show ntp associations

    # YAMAHA側の設定
    schedule at 1 */* 00:00 * command ntpdate 172.23.1.2