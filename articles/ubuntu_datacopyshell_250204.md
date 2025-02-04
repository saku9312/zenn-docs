---
title: "Ubuntu上でHDD1→HDD2への自動ミラーリング"
emoji: "🐿️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## やりたいこと
HDDを2台繋いでいて、1台をメイン、1台をバックアップとして、1日1回自動でミラーをしてほしい。
Ubuntu24.04上で実行するように設定した。

## shellをcronで定期実行
```
#backup.shの作成
sudo vi backup.sh
    rsync -avud -delete /mnt/hdd1 /mnt/hdd2

#cronで定期実行（nano選択）
sudo crontab -e
1
#毎日3時に実行
0 3 * * * backup.sh > /var/log/cron.log 2>&1

#テスト実行
sudo bash backup.sh
```
## 使用しているshellのオプション
-a アーカイブモード：ファイル属性を保持
-v 詳細モード：経過表示
-u 更新モード：宛先のファイルが新しい場合はコピーしない
-z 圧縮モード：転送データを圧縮し、帯域幅を節約
-delete ：宛先に存在し送信元に存在しないファイルは削除する（完全ミラー）
2>&1：1が標準出力で2がエラー出力。この場合はエラー出力も標準出力にいれて吐き出してと設定している