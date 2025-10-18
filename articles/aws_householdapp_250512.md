---
title: "AWSのEC2を使わずに家計簿アプリを作ってみた"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに
AWSのSAAを取得するために勉強を始めたが、いかんせんAWSの習熟度が低く知識が頭に入ってこない。
無料枠がまだ残っていたので、EC2を使わずに家計簿アプリを作成し、動きを勉強してみる。

# 使用サービス
- AWS Amplify
- AWS Lambda
- AWS VPC
- AWS API Gateway
- AWS RDS(Postgre SQL)
- AWS CloudShell
- GitHub

# 言語
- Python 3.9
- HTML/CSS/JavaScript（AI使用）

## 概略
おおまかに流れを作ると、

1. IAMでユーザーの作成
2. RDSからPostgreSQLを使ったデータベースを作成
3. Lambda関数でPythonを使ったコードを作成
4. API Gatewayのセットアップ
5. Amplifyを使い、フロントエンドをホスティングする
6. ブラウザからテストし、データベースへ値の投入ができるかを確認する

今回、素人なのでセキュリティ面は今後学ぶものとし、
最低限動作を確認できるところまで確認できればOKとしたい。

## IAMユーザーの作成
今のところデフォルトで作成されたユーザーしか存在しないため、新規ユーザーを作成する。

1. 

## RDSを使用したデータベースの作成
今回はPostgreSQLを使用したデータベースを作成し、Lambda関数で読み書きできるように設定する。
データベースはかじった程度なのでとりあえず動くように。

1. 

## Lambda関数を作成
Pythonのpsycopg2ライブラリを使用して、データベースへアクセスできるようにしたい。
ただ、Python3.8以上はAWS上のライブラリがサポートされておらず、CloudShellを使用して、パッケージを作成している。
