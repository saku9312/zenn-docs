---
title: "YAMAHAとCiscoのNTP設定"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# CiscoとYAMAHAでのNTP設定
宅内環境でNTPサーバーを立てたので、そこから時刻同期をするためにNTPの設定を各ネットワーク機器に投入する。
YAMAHAはスケジュール組んでやらないと一回きりになるので注意。

## ciscoでNTPの同期設定

    ntp server サーバーIP
    show ntp status
    show ntp associations

## YAMAHAでNTPの同期設定

    ntpdate サーバーIP
    show environment
    #ntpdateだけだと1回しか同期しないためスケジュールを組む
    schedule at 1 */* 01:00 * ntpdate サーバーIP

## 追加情報
・ 1はスケジュール番号(任意)
・ */*は日付指定(月/日) */で毎日という意味
・ 01:00は時刻の指定(hh:mm)
・ 01:00 * command間のは必須入力
・ commandは実行するコマンド指定