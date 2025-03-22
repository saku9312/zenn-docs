---
title: "構成デザイナーでWindowsPCのセットアップ"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに
前職でSysprepによるキッティングを試したが、導入まで至らずに悶々としていた。
色々と見ていたところ、「構成デザイナー」による自動キッティングが簡単そうで、しかも自動セットアップ可能と分かったので自宅環境で試してみる。

## 構築環境
- Windows11 24H2
- Windows Server 2025 ActiveDirectory
- USBメモリ
- 回復済み または セットアップ済みのWindows11 PC

## 構成デザイナーのインストール
構成デザイナーは以下のサイトからダウンロードして、Windows ADKの機能の選択からインストールすることになる。
https://learn.microsoft.com/ja-jp/windows-hardware/get-started/adk-install#download-the-adk-for-windows-11-version-24h2

1. 上記サイトからWindowsADKのインストーラーをダウンロードする
![](https://storage.googleapis.com/zenn-user-upload/2f4472376c60-20250322.png)

2. インストーラーを実行する
利用規約やディレクトリの指定だけして、機能の選択画面で「イメージングおよび構成デザイナー」が選択されていることを確認する。
![](https://storage.googleapis.com/zenn-user-upload/8e4dcd317109-20250322.png)

3. 構成デザイナーを起動し、デスクトップのプロビジョニングを開始
スタートメニュー内の「Windows Kit」から起動し、デスクトップデバイスを選択する。
![](https://storage.googleapis.com/zenn-user-upload/e5ff5ffdde46-20250322.png)

4. プロジェクトの保存先を指定する
![](https://storage.googleapis.com/zenn-user-upload/ff37f9173f22-20250322.png)

5. デバイスのセットアップ画面
PC名を自動で割り振ることやプロダクトキーの埋め込みが可能。
%SERIAL%でシリアルを、%RAND:5%でランダム5桁を自動で割り振る。
今回は頭に「saku-」をつけ、それ以降を3桁のランダム文字とした。
![](https://storage.googleapis.com/zenn-user-upload/f7844ce495d0-20250322.png)

6. ネットワークのセットアップ画面
接続したいWi-Fiの設定をあらかじめ投入できる。
企業側で導入している無線ネットワークがあれば、ここで自動認識させられる。
![](https://storage.googleapis.com/zenn-user-upload/7a0a2d9b90eb-20250322.png)

7. アカウント管理画面
あらかじめActive Directoryに参加させたり、ローカルアカウントの作成ができる。
今回は検証用のADに参加させてみる。
![](https://storage.googleapis.com/zenn-user-upload/cae0c060efa9-20250322.png)

8. アプリケーションの自動インストール設定
自動的にインストールしておきたいアプリを共有フォルダに入れておけば、セットアップ時に自動で入れてくれる。
今回は試しにAdobeReaderのインストーラーを置き、ドメイン上の場所から実行できるか試してみる。

＜注意＞
「追加」を押さないとアプリが登録されません。

![](https://storage.googleapis.com/zenn-user-upload/541bf91f9b65-20250322.png)

9. 設定内容を保存する
すべての設定が完了したら、作成をクリックする。
![](https://storage.googleapis.com/zenn-user-upload/8ca999459dab-20250322.png)

10. パッケージファイルが作成されたことを確認する
作成が完了すると下部にフォルダパスが表示され、クリックするとプロジェクトのあるフォルダに移動する。
今回はこのProject_1.ppkgとProject_1.cat を使用する。
![](https://storage.googleapis.com/zenn-user-upload/4b05f8e02359-20250322.png)

11. パッケージファイルをUSBメモリに移動する
作成したパッケージファイルは、USBメモリのルートディレクトリにコピーする。

＜注意＞
どこかのフォルダに入れたりすると、自動セットアップ時にパッケージファイルが認識されません。

ここまでで、自動セットアップ用のパッケージファイルの準備が完了した。
次に自動セットアップを試してみる。

## パッケージファイルを使用した自動セットアップ
前提として、回復済みのPCを使用する。
セットアップ画面でUSB挿したら即動き出してびっくり。

1. セットアップしたいPCに、パッケージファイルを入れたUSBを接続して起動する

2. 自動的にパッケージファイルが認識されて、キッティングが開始される。

3. 画面を確認すると、Adobe Readerが自動的にインストールされ、ドメイン参加もされていた。

# まとめ
前職の時にしっかり検証できればよかったのだが、他業務に忙殺されてまったく手を付けられず。
もやもやしていたので、じっくり自宅で検証できて良かった。
キッティングの手数で悩まれている情報システムの方はまだいると思うから、こういった形で提案していければより他の業務に時間をかけられると思う。
