---
title: "blenderの自分がよく使うショートカットキー"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
CiscoのAironet2800を入手して喜んでいたが…、わかっていたけれど、K9で終わる型番は通常ではMobilityExpressが使用できないので、何とか使える形まで持って行った。

## Cisco AironetのMobility Express化
ファームウェアの変更が必要で、法人のアドレスを使う必要があるので注意。
デフォルトのID・パスワードはCisco。
あまりファームウェアを上げすぎたせいか、一度失敗したのだが、日をあけて再度試したら成功した。

    #tftpサーバーにファームウェアを入れて置き、インストール実施
    archive download-sw /reload tftp://172.23.1.2/AIR-AP2800-K9-ME-8-10-196-0.tar

もし成功したら、以下の手順でGUIから設定。
1. 「CiscoAireProvision」というSSIDに接続
2. 自動的にブラウザからアクセスされるので、IDとパスワードを決定（CUIの初期パスはpassword）
3. SSIDの設定は、WLANから行う
4. アクセスポイントの選択から、IPを固定しておく
5. DHCPサーバーは無効にする

## APの追加
APを追加する場合は、バージョンを統一して（もちろんMEのもので）同ネットワークに接続すると、Controller側で自動的に認識する。