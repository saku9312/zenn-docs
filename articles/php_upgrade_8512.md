---
title: "ポートフォリオサイト（WordPress）のPHPアップデートに急ぎ対応したログ"
emoji: "🐾"
type: "idea"
topics: ["upgrade", "wordpress", "php", "脆弱性"]
published: true
---

# はじめに

10/21にPHPの脆弱性が発表されました。

https://www.php.net/

使用中のポートフォリオサイトがこの対象になっているため、脆弱性対応としてアップデートを行ったので、
記録として残しておきたいと思います。
（対応時間 10/22 6時）


# PHPのバージョン確認

``` bash

:php -v

```
コマンドを実行した結果、アップグレードが対象まで上がっていないため、
Ubuntuのリポジトリよりアップデートを取得し、アップグレードを実行しました。

```bash

:apt update && upgrade -y

Setting up php8.5-cli (8.5.12-1+ubuntu24.04.1+deb.sury.org+1) ...
Setting up php8.5-phpdbg (8.5.12-1+ubuntu24.04.1+deb.sury.org+1) ...

～～その他省略～～

```

今回の脆弱性は危険度も高く、各所から迅速にアップグレードするようにアナウンスされていたのが印象的でしたね。

★ IPA
https://www.ipa.go.jp/security/security-alert/2026/pre-alert20261021.html

★ 高市総理のX
https://x.com/takaichi_sanae/status/2112890642590564386


それに加えて、多数のサイトが本件の対応のためにサイトを一時的に閉鎖し、
公開時間の23時を目途にアップデート対応をする旨をアナウンスされていたのも印象的でした。


また、今回の概要は以下の通りですが、自分のサイトが踏み台にされると危険なため、迅速に対応が必要と判断しています。

• CVE番号: CVE-2026-109618（メモリ不足処理に関する脆弱性）、CVE-2026-109619（解放済みメモリへの参照が残る脆弱性）
• 深刻度: 最高評価の Critical（CVSS 4.0基本値: 9.2）
• 主なリスク: 認証なしのリクエストにより、外部からサーバー上で任意のコードを実行され、サイト乗っ取りや情報漏えいにつながる恐れがあります
• 影響を受ける構成: PHP-FPM、Apacheのmod_php、php-cgi、FrankenPHP、LiteSpeedなど（特にmod_phpは危険性が高い）、設定変更やWAFでも防御不可！


アップデートは完了したので、バージョンを確認してOKか確認します。

```bash

:php -v

PHP 8.5.12 (cli) (built: Oct 21 2026 14:37:30) (NTS)
Copyright (c) The PHP Group
Built by Ubuntu
Zend Engine v4.5.12, Copyright (c) Zend Technologies
    with Zend OPcache v8.5.12, Copyright (c), by Zend Technologies

```

大丈夫そうですね。ちゃんとパッチ対応されたバージョンに上がっています。

実行中も同じリポジトリに多くのマシンからアクセスされているのか、
2回ぐらいアップデートに失敗しました。

```bash

Failed to fetch https://ppa.launchpadcontent.net/ondrej/php/ubuntu/pool/main/p/php8.5/php8.5-phpdbg_8.5.12-1%2bubuntu24.04.1%2bdeb.sury.org%2b1_amd64.deb  503  Service Unavailable [IP: 185.125.189.187 443]

```

自分のサイトは緊急性がないので申し訳ないところではありましたが、対応が完了したので良しとします。

多くの企業サイト管理者および担当者の方々は、遅くまで対応されていたのではないでしょうか。お疲れ様でした。。。