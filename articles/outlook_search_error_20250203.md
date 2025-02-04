---
title: "Outlook2016で検索結果の差出人表示ができない"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## 発生した環境
- Windows11 24H2
- Outlook 2016

## 症状
Outlook上でメールの検索を行った場合に、差出人の表示だけ真っ白になる。
また、なぜか閲覧済みのメールもすべて含めて未読になってしまう。

## 対処
レジストリの値の追加で解決。
- パス：HKEY_CURRENT_USER\software\policies\Microsoft\office\16.0\outlook\search
- キー名：DisableServerAssistedSearch
- 種類：REG_DWORD
- キー値：1

    @echo off
    reg add "HKEY_CURRENT_USER\software\policies\Microsoft\office\16.0\outlook\search" -v DisableServerAssistedSearch /t REG_DWORD /d 1
    pause