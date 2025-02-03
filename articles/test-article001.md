---
title: "ZennとGitHubの連携をやってみる"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# ZennとGitHubの連携
記事投稿をしないとと思い、改めてGitHubとの連携設定を確認。

## 手順
以下の手順が必要なので、忘れずにやる。
公式には完全初心者の自分のような部分の漏れがある。

1. GuiHub Desktopのダウンロードし、リポジトリの作成を行う
2. ディレクトリにクローンを行う
3. Zennから連携を行う
4. Node.jsをインストールする
5. Node.js Command Promptから、クローンしたディレクトリに移動してURLに記載の手順からZenn CLIをインストールする
6. 記事の新規作成は、毎回Node.jsのコマンドプロンプトから実施する
7. articleには何も内容を入れないとエラーが発生するので注意

https://zenn.dev/zenn/articles/zenn-cli-guide
http://zenn.dev/zenn/articles/install-zenn-cli

## とりあえず
Visual Studio Codeのインストールおよび日本語化は今回触れないが、そのまま調べてやれば問題なし。（拡張機能でJapaneseと検索すればいい）

Node.jsのディレクトリ移動はどうするの？など最初に躓く点があったが、何とか記事の投稿はできている。