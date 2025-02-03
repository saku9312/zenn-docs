---
title: "Proxmoxの環境構築"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# Proxmoxの環境構築
Raspberry Pi 4BにProxmoxを入れてみる。
想定する構成はRaspberry Pi 4B 3台を使用。
https://remoteroom.jp/diary/2024-05-01/

### OSのインストールと下準備
今回はRaspberry Pi OS Lite（64-bit）を使用。

1. Raspberry Pi ImagerでmicroSDにISO書き込み
2. ネットワークの固定（ラズパイはREDHAT系のnmcliコマンドで固定可能）

       #デフォルト状態を確認
       nmcli connection
       #名称変更
       nmcli connection modify 'Wired connection 1' connection.id 'eth0'
       nmcli connection
       #現在のeth0設定を確認
       nmcli -f ipv4 connection show eth0
       #IPやゲートウェイの設定を投入
       nmcli connection modify eth0 ipv4.addresses 172.23.1.3/24
       nmcli connection modify eth0 ipv4.gateway 172.23.1.254
       nmcli connection modify eth0 ipv4.dns 172.23.1.9
       nmcli connection modify eth0 ipv4.method manual
       nmcli connection up eth0
       #設定が反映されているか確認
       nmcli -f ipv4 connection show eth0

3. ホスト名とIPアドレスを紐づける

       #/etc/hostsファイルの編集
       127.0.1.1 saku-rb1 →コメントアウト
       172.23.1.3 saku-rb1
   
4. LXC Containerのmemory/swap情報を正しく反映させるため、以下のパラメータを追記

        #/boot/firmware/cmdline.txtを編集
       vi /boot/firmware/cmdline.txt
           #最終行に追記
           cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1

5. kernelを指定のタイプへ変更する

        #/boot/firmware/config.txtを編集
       vi /boot/firmware/config.txt
           #最終行に追記
           kernel=kernel8.img
       #リリース番号を確認した後、OSを再起動
       uname -r
       shutdown -r now
       #リリース番号が変更されていることを確認
       uname -r
       6.6.67+rpt-rpi-v8

### PVEのインストール
1. Proxmox-Portのリポジトリを登録し、apt情報を登録、導入済みパッケージを最新化する→apt実行時に参照できるようにする

        #/etc/apt/sources.list.d/pveport.listに追記
       #echo 'deb [arch=arm64] https://mirrors.apqa.cn/proxmox/debian/pve bookworm port' >/etc/apt/sources.list.d/pveport.list
       #アクセス確認
       #curl -L https://mirrors.apqa.cn/proxmox/debian/pveport.gpg -o /etc/apt/trusted.gpg.d/pveport.gpg
       sudo apt update
       sudo apt full-upgrade

2. PVEのパッケージをインストールする

        sudo apt install ifupdown2
        sudo apt install proxmox-ve postfix open-iscsi

3. postfixのインストール画面が表示されるので、以下の設定で進める
    ・「General email configuration type:」→「Local Only」
    ・「System mail name」→そのまま
    ・「Configuration file '/etc/apt/sources.list.d/pveport.list'」→「Y or l :install the package maintainer's version」

4. OS再起動し、rootユーザーのパスワードを変更

        sudo -s
        passwd
        →設定したら完了

### PVEの管理画面アクセスと準備
1. 設定したIP（起動時に上部へポート番号も表示される）にアクセス
2. ネットワークの設定を追加
   ・ノード名＞システム＞ネットワーク　から、作成をクリック
   ・「Linux Bridge」を選択し、「vmbr0」に設定を追加する。
   ・ポートは「eth0」、CIDRに設定したIP（今回は172.23.1.3/24）、ゲートウェイを入力
   ・Linux Bridgeは物理NICに紐づけるもので、複数NICがあれば複数設定できる
   ・これから作成するインスタンスのIPは、起動後にネットワークタブから設定するので注意！　ここで別のIPを設定すると、そのIPで管理画面にアクセスすすことになる

### 無償版リポジトリに変更する
初回起動時はエンタープライズ契約向けになっているため、設定を変更する。
1. ホスト名＞アップデート＞リポジトリ　にアクセス
2. https://enterprise.proxmox.com/debian/pve と書かれたものをクリックして選択し、 無効ボタンを押して無効化
3. 先ほどの無効ボタンの左横の追加ボタンを押し、表示されたダイアログ上で No-Subscription を選択して追加

### テンプレートのダウンロード
インスタンス起動のためにイメージが必要になるため、テンプレートをダウンロードしておく。
1. local＞CTテンプレート＞テンプレート　から、使いたいイメージをダウンロードする（今回はARMのみ）
2. ダウンロードすると、インスタンス作成時に使える

### コンテナの作成
1. 「CTの作成」をクリック
2. ホスト名、パスワードを入力（rootに対するパスワードになる）
3. テンプレートイメージの選択
4. ディスクサイズを選択
5. CPUコア数、メモリとスワップ領域を指定
6. ネットワークは使用する物理NICを選択（NIC1つだけならeth0とvmbr0になる）、DHCPや固定IPの設定
7. dnsドメインとサーバーIPの指定（saku.com、172.23.1.9）で設定
8. 作成をクリックする

### コンテナの利用
ホスト名の下にコンテナが作成するため、コンソールタブからアクセスが可能。
問題がなければ外部から設定したコンテナへのPingなども通る。

### CTとVMの違い
VM→Windowsなどが乗せられるが、容量とスペックが必要。
CT→Linuxは基本こっちで動かす、高速だがカーネル部分の操作が不可能。

## 複数台のProxmoxでクラスターの設定
残り2台をセットアップして、クラスターを作っていく。
最終的にはk8sで設定したい！

### クラスター前準備
以下の項目をそろえる。
・時刻・時間が同期されている　→chronyで統一
・rootユーザーのパスワードが設定されている　→passwdで設定
・HA機能を利用する場合は、3台以上のノードがクラスターに参加している　→先述の通り3台立てればよい

### クラスター設定開始
3台のセットアップが完了したため、クラスターを作成。

1. クラスター＞クラスター作成をクリック
2. クラスター名を指定し、自分のノードが選択されている状態で進める
    ※クラスター名は作成後変更できない、またこの時点では他ノードは選択しない
3. 作成後はクラスターをクリックし、コピーで設定をコピーする
4. 追加したいノードにアクセスし、クラスター＞クラスターに参加をクリック
5. コピーした設定を貼り付け、親になるノードのパスワードと対象となるマシンを選択
6. 開始すると少し時間がかかるが、問題なく追加されるとどのノードのGUIからもクラスター内の構成が確認できるようになる。
    ※2回やって2回ともエラーが一旦出たが、親ノードからすべて確認ができたし、CTやVMの作成もできた。問題はなさそう？

## ProxmoxHAの設定
データセンター内でHAの設定をかけておくことで、対象のVMやコンテナに障害が発生した際に別ノードで自動起動させることが可能。

1. データセンターを選択
2. HAの項目へ移動
3. 対象のVMやコンテナを選択
4. グループを作成させると、選択したノードごとにプライオリティを付けることも可能
5. 追加をすればHAは完成