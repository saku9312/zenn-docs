---
title: "LinuxでHDDをマウントする"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
Linuxでは、パーティションの作成とパーティションのフォーマットは別の作業が悲痛になる。
fdiskでパーティションの作成、mkfsでパーティションのフォーマットを行う。
この両方をやらないとドライブとして使えない。

## 外付けHDDのマウントとフォーマット

    # 外付けHDDの確認
    fdisk -l
    # フォーマット
    umount -a /dev/sda
    mkfs -t ext4 /dev/sda
    # UUIDの確認
    blkid /dev/sda
    # マウントディレクトリの作成
    mkdir /mnt/hdd
    chmod 744 /mnt/hdd
    # マウントの記載
    vi /etc/fstab
        UUID=<uuid> /mnt/loghdd/rsyslog ntfs defaults 0 0
    # 再起動して確認
    shutdown -r now
    # うまくいかなかったら、以下で調整
    mount -a
    df -h
        
## Q.すでにデータが入っているディレクトリに、HDDをマウントした場合どのような挙動をする？

A.マウントされたHDDのデータは閲覧できるが、もともとそのディレクトリに入っていたデータは閲覧できなくなる。
今回はデータが入った状態で誤ってマウント→どうしても容量が何やっても減らない→原因はもともとディレクトリに入っていたファイルの容量が大きかった、ということが発生。

ディレクトリパスも、マウントしたものと統合され存在自体が見えなくなってしまうため、データがないことを確認してからマウントしましょう…。
