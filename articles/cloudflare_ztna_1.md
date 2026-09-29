---
title: "【第1回】CloudflareとGWSを使用してゼロトラストを検証してみた　～DNS設定＆Cloudflare Tunnelの設定編～"
emoji: "🔒"
type: "tech"
topics: ["cloudflare", "wordpress", "proxmox", "ztna", "dns", "gws"]
published: true
---
# はじめに
Geminiでスプレッドシートを自動生成させたいと思い、いい機会なのでGWSを契約してみました。
そのとき、「Googleアカウントを使って自宅サーバーにゼロトラストアクセスを構築できるのでは？」と思い、Cloudflare Tunnelと組み合わせて検証してみました。

いろいろと検証をしてみたところ、DNSの移管がそもそも必要だったり、GWSの無料枠でやるにはアカウントの作り方を考える必要があったり、
SSOではなくゼロトラストを構築するにはCloudflare Accessの設定が必要だったりと、いろいろとハマるポイントがありました。

第1回となる本記事では、**お名前.com から Cloudflare への DNS 移行**、および **Cloudflare Tunnel (`cloudflared`) を用いたインバウンドポート完全閉鎖環境での Web 公開** までを解説します。

---

## 1. アーキテクチャ構成と仕組み

今回構築するインフラの全体像は以下の通りです。

```text
[ クライアント (ブラウザ) ]
       │  (HTTPS: 443)
       ▼
[ Cloudflare Edge Network ] ── (DNS / WAF / Proxy)
       │
       │  ★ インバウンドポート解放不要！
       │  ★ アウトバウンドの QUIC / gRPC トンネル（暗号化）
       ▼
[ 自宅 LAN / Proxmox (TurnKey Linux / Debian) ]
       └─ [ cloudflared デーモン ] ── (HTTP: 80) ──> [ WordPress (LXC/VM) ]
```

### なぜ Cloudflare Tunnel なのか？

Cloudflare Tunnel（旧 Argo Tunnel）は、自宅サーバー側で動作する軽量コネクター（`cloudflared`）が、Cloudflare のエッジサーバーに対して**内側から外側へ（アウトバウンド）暗号化トンネル（QUIC または TCP）を自動確立**する技術です。

1. **インバウンドポート開放不要:** ルーターで 80 番や 443 番ポートを開ける必要が一切ありません。ファイアウォールは「外部からの全通信拒否」のままで機能します。
2. **IP アドレスの隠蔽:** ブラウザからのリクエストはすべて Cloudflare のプロキシ IP アドレスで終端されます。自宅のグローバル IP アドレスが外部に漏れることはありません。
3. **CGNAT / IPv4 枯渇問題の回避:** アウトバウンド通信さえ確立できれば動作するため、IPv4 グローバル IP が割り振られない環境でもパブリック公開が可能です。

---

## 2. 費用・前提条件・制限事項

本構築を進める前に、費用と動作要件、および無料プランにおける制約を整理します。

### 費用・要件一覧

| 項目 | 条件 / 費用 | 補足 |
| :--- | :--- | :--- |
| **独自ドメイン** | 年間数千円程度（実費） | お名前.com、Cloudflare Registrar 等で取得 |
| **Cloudflare アカウント** | **完全無料**（Free プラン） | Zero Trust 枠（50 ユーザーまで無料）を使用 |
| **自宅サーバー** | Proxmox VE / Linux / Docker等 | 本記事では Debian 系（TurnKey Linux）を使用 |
| **ネットワーク環境** | インターネット接続環境 | ポート開放不要、固定 IP 不要 |

### Cloudflare Free プランの制限事項・注意点

1. **利用規約（Self-Serve Subscription Terms / Section 2.8）の遵守**  
   Cloudflare Tunnel を経由して**大容量の動画ファイルや非 HTML コンテンツ（動画ストリーミング等）を大量配信する場合**、無料プランでは利用規約に抵触する恐れがあります。通常の WordPress ブログや Web API 運用であれば全く問題ありません。
2. **クォータと帯域幅**  
   Tunnel 自体の帯域上限は明示されておらず、個人〜小規模運用であれば実用上十分なパフォーマンスが得られます。

---

## 3. お名前.com から Cloudflare への DNS 移行

まずは、ドメインの権威 DNS サーバーをお名前.com から Cloudflare へ移管（委任）します。

### Step 1: Cloudflare にドメインを追加

1. [Cloudflare ダッシュボード](https://dash.cloudflare.com/) にログインし、画面右上の **[サイトを追加]** をクリックします。
2. お名前.com で取得済みの独自ドメイン（例: `example.com`）を入力します。
3. プラン選択画面で **[Free ($0)]** を選択して **[続行]** をクリックします。

![](https://static.zenn.studio/user-upload/22d1a98e843c-20260929.png)

4. 既存の DNS レコードが自動スキャンされます。既存の A レコードや MX レコード（メールサーバー設定）が検出されていることを確認し、**[続行]** をクリックします。

> **⚠️ 超重要：メール（MX レコード）の移行漏れ注意**  
> 既存ドメインでメール（Google Workspace、独自メールサーバー等）を運用している場合、ここで MX レコードや SPF（TXT）レコードが正しく読み込まれているかを**必ず目視確認**してください。ネームサーバー切り替え後にメールが届かなくなる原因の 9 割がこの設定漏れです。

5. Cloudflare から割り当てられた **2 つのネームサーバー（NS）**（例: `ada.ns.cloudflare.com`, `bob.ns.cloudflare.com`）をメモします。

---

### Step 2: お名前.com 側でネームサーバーを変更

1. [お名前.com Navi](https://www.onamae.com/navi/login/) にログインします。
2. ネームサーバー設定メニューから **[ネームサーバーの変更]** を開きます。
3. 対象のドメインを選択し、**[他のネームサーバーを利用]** タブに切り替えます。
4. **1 枠目** と **2 枠目** に、先ほど Cloudflare で発行されたネームサーバーアドレスをそれぞれ入力し、設定を保存します。

![](https://static.zenn.studio/user-upload/97edbaafed0e-20260929.png)

---

### Step 3: DNS 反映確認とトラブルシューティング

ネームサーバーの変更には、世界中の DNS キャッシュの更新に伴い **数分〜最大 24 時間** かかります。

#### 反映状況の確認コマンド (CLI)
ローカルのターミナル（Mac / Linux / Windows WSL）から `dig` や `nslookup` を実行し、NS レコードが Cloudflare のものに切り替わっているか確認します。

```bash
nslookup test-wp.example.com
# 出力例:
# ada.ns.cloudflare.com.
# bob.ns.cloudflare.com.
```

Cloudflare ダッシュボード上で **「アクティブ」** 表示になれば、DNS 移行は成功です。

---

## 4. Proxmox (TurnKey Linux) 環境のセットアップ

本構築では、**Proxmox VE** 上に構築された **TurnKey Linux WordPress (LXC コンテナ)** を対象とします。（※通常の Debian / Ubuntu / Docker 環境でも手順は全く同一です）

### ネットワーク前提条件

* **WordPress サーバーのローカル IP:** `192.168.10.50`（例）
* **OS:** Debian 11 / 12 ベース
* **必須パッケージ:** `curl`, `gnupg`, `systemd`

サーバー側で事前に静的 IP が割り当てられており、ローカルネットワーク内から `http://192.168.10.50` で WordPress の初期画面が開くことを確認しておきます。

---

## 5. Cloudflare Tunnel (`cloudflared`) の作成と接続

次に、Cloudflare Zero Trust ダッシュボードでトンネルを作成し、WordPress サーバー上で `cloudflared` コネクターを常駐プロセス（systemd サービス）として起動します。

### Step 1: Zero Trust ダッシュボードで Tunnel を定義

1. [Cloudflare Zero Trust ダッシュボード](https://one.dash.cloudflare.com/) にアクセスします。
2. 左サイドメニューから **[Networks] > [Tunnels]** を選択します。
3. **[Add a tunnel]**（トンネルの追加）をクリックします。
4. トンネルタイプとして **[Cloudflared]** を選択し、**[Next]** をクリックします。
5. トンネル名（例: `wp-tunnel`）を入力し、**[Save tunnel]** をクリックします。

![](https://static.zenn.studio/user-upload/0c6797db9ca7-20260929.png)

---

### Step 2: WordPress サーバーへ `cloudflared` をインストール

トンネルを作成すると、OS ごとのインストールコマンドとアクセストークン（`eyJh...`）が自動生成されます。

Proxmox の TurnKey Linux（Debian ベース）コンテナに SSH ログインし、以下のコマンドを実行します。

#### 1. リポジトリの登録と `cloudflared` のインストール

```bash
# 必要なパッケージのインストール
sudo apt-get update && sudo apt-get install -y curl gnupg

# Cloudflare GPG キーの追加
sudo mkdir -p /usr/share/keyrings
curl -fsSL [https://pkg.cloudflare.com/cloudflare-main.gpg](https://pkg.cloudflare.com/cloudflare-main.gpg) | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

# APT リポジトリの追加
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] [https://pkg.cloudflare.com/cloudflared](https://pkg.cloudflare.com/cloudflared) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-main.list

# パッケージ更新と cloudflared のインストール
sudo apt-get update && sudo apt-get install -y cloudflared
```

#### 2. トンネルサービス（systemd）の登録と起動

ダッシュボードに表示されたトークンを使用して、`cloudflared` をシステムサービスとして登録・起動します。

```bash
sudo cloudflared service install <あなたのTOKEN文字列>
```

> **解説:**  
> このコマンドを実行すると、`/etc/systemd/system/cloudflared.service` が自動生成され、OS 起動時に `cloudflared` デーモンが背景で常駐するように設定されます。

#### 3. サービス状態の確認

```bash
sudo systemctl status cloudflared
```

**【出力例】**
```text
● cloudflared.service - cloudflared
     Loaded: loaded (/etc/systemd/system/cloudflared.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2026-09-28 10:00:00 JST; 10s ago
   Main PID: 12345 (cloudflared)
      Tasks: 9 (limit: 4915)
     Memory: 25.4M
        CPU: 120ms
     CGroup: /system.slice/cloudflared.service
             └─12345 /usr/bin/cloudflared --no-autoupdate ready --token eyJh...
```

ダッシュボード上でも、Status が **`HEALTHY`**（緑色）に変化したことを確認してください。

---

### 高利用率・冗長化（マルチコネクター）について

Cloudflare Tunnel は、**1 つの Tunnel ID に対して複数のコネクター（`cloudflared`）を同期接続** させることが可能です。

たとえば、Proxmox 上の別ノードや、別の物理 PC にも同じトークンで `cloudflared` を起動しておくと、片方のサーバーがメンテナンス等でダウンしても、Cloudflare エッジが自動的にもう一方の正常なコネクターへ通信をルーティングしてくれます。これが**完全無料**で構築できる点も大きなメリットです。

---

## 6. Public Hostname (Ingress Rules) の設定とプロキシの仕組み

トンネルが開通したら、外部からのパブリックドメイン（例: `test-wp.example.com`）のリクエストを、ローカルの WordPress（`http://192.168.10.50:80`）へ転送するルール（Ingress Rules）を設定します。

### Step 1: Public Hostname の追加

1. Zero Trust ダッシュボードの Tunnel 設定画面で、作成した Tunnel の **[Public Hostname]** タブを開きます。
2. **[Add a public hostname]** をクリックします。
3. 以下の通り設定します：

| 項目 | 設定値 | 説明 |
| :--- | :--- | :--- |
| **Subdomain** | `test-wp` | 任意のサブドメイン |
| **Domain** | `example.com` | Cloudflare に登録したドメイン |
| **Path** | （空欄） | ルートパス全体を指定 |
| **Type** | `HTTP` | ローカル Web サーバーのプロトコル |
| **URL** | `192.168.10.50:80` | WordPress サーバーのローカル IP とポート |

![](https://static.zenn.studio/user-upload/7133bb59a925-20260929.png)

4. **[Save hostname]** をクリックします。

この保存操作により、Cloudflare の DNS テーブルに `test-wp.example.com` の **CNAME レコードが自動生成**され、Tunnel ID（`<TUNNEL_ID>.cfargotunnel.com`）へルーティングされます。

※DNSレコードが自動生成されない場合、以下の通りにレコードを設定します。
![](https://static.zenn.studio/user-upload/40819a8c54d1-20260929.png)

---

### Step 2: DNS レコードと「プロキシ状態 (Proxy Status)」の確認

Cloudflare ダッシュボードの **[DNS] > [Records]** を開くと、以下のような CNAME レコードが自動で追加されていることがわかります。

```text
Type: CNAME
Name: test-wp
Target: <TUNNEL_ID>.cfargotunnel.com
Proxy status: Proxied (オレンジの雲)
```

登録されていない場合は、手動で CNAME レコードを追加してください。

#### 「オレンジの雲 (Proxied)」と「グレーの雲 (DNS Only)」の違い

Cloudflare を使用する上で最も重要な概念が、この Proxy Status（雲の色）です。

```text
【 Proxied (オレンジの雲) 】★ 必須
  [ User ] ──(HTTPS)──> [ Cloudflare Edge (WAF/Proxy) ] ──(Tunnel)──> [ Server ]
  ・Cloudflare の WAF、SSL 終端、DDoS 防御、Access (ZTNA) 機能が「有効」になる。
  ・クライアントには Cloudflare の IP のみが返る。

【 DNS Only (グレーの雲) 】
  [ User ] ─────────────── Direct DNS Resolution ───────────────> [ Server ]
  ・Cloudflare は単なる DNS サーバーとして機能する。
  ・プロキシや WAF、Cloudflare Tunnel 経由のアクセスは利用できない。
```

Cloudflare Tunnel を使用する場合は、**必ず「Proxied (オレンジの雲)」を有効**に維持する必要があります。

---

### Step 3: SSL/TLS 暗号化モードの設定

Cloudflare エッジとブラウザ間の通信を適切に暗号化するために、Cloudflare ダッシュボードの **[SSL/TLS] > [Overview]** で暗号化モードを設定します。

* **Full (strict)** （推奨）: ローカル Web サーバー側で自己署名証明書（SSL）を有効にしている場合。
* **Flexible**: ローカル Web サーバー側が HTTP (ポート 80) のみで動作している場合。

本環境ではローカル WordPress が HTTP (ポート 80) で動作しているため、**[Flexible]** または **[Full]**（HTTP 通信をそのままトンネリング）を選択します。

---

## 7. 動作検証と WordPress 特有の問題回避

設定が完了したら、実際にブラウザから `https://test-wp.example.com` へアクセスします。

![](https://static.zenn.studio/user-upload/6a949fe0928a-20260929.png)

自宅ルーターのポートを 1 つも開けていない状態にもかかわらず、Cloudflare エッジ経由で世界中から安全にアクセスできるようになりました。

---

### WordPress リダイレクトループ（`TOO_MANY_REDIRECTS`）の対処法

Cloudflare Tunnel 経由で WordPress を公開した際、**「リダイレクトループ（`ERR_TOO_MANY_REDIRECTS`）」** が発生することがよくあります。

#### 原因
1. ブラウザと Cloudflare 間は **HTTPS (443)** で通信している。
2. Cloudflare と WordPress（ローカル）間は **HTTP (80)** で通信している。
3. WordPress はリクエストが HTTP だと判断し、HTTPS へリダイレクト（`301`）しようとする。
4. これが無限ループに陥る。

#### 解決策
WordPress の `wp-config.php` の先頭（`<?php` の直下）に以下のコードを追加し、Cloudflare 側の HTTPS 判定を WordPress に認識させます。

```php
// Cloudflare からの HTTPS リクエスト判定を正常化
if (isset($_SERVER['HTTP_CF_VISITOR']) && strpos($_SERVER['HTTP_CF_VISITOR'], 'https') !== false) {
    $_SERVER['HTTPS'] = 'on';
}

// サイト URL の固定設定（必要に応じて）
define('WP_HOME', '[https://test-wp.example.com](https://test-wp.example.com)');
define('WP_SITEURL', '[https://test-wp.example.com](https://test-wp.example.com)');
```

また、アクセスのアクセス元 IP アドレスを正確にログ記録するため、必要に応じて WordPress のプラグインまたは Web サーバー（Nginx / Apache）側で `CF-Connecting-IP` ヘッダーを読み込む設定を追加してください。

---

## 8. まとめと次回予告

第1回では、ゼロトラストな自宅サーバー環境の「土台」となる以下のステップを完遂しました。

1. **お名前.com から Cloudflare への DNS 移行:** 安全かつ高機能な権威 DNS の確保。
2. **`cloudflared` による暗号化トンネルの確立:** ポート開放不要で自宅 LAN と Cloudflare エッジを接続。
3. **Public Hostname のマッピング:** プロキシ（オレンジの雲）を経由した安全な Web 公開。

現在の状態では、トップページ（`/`）も管理画面（`/wp-admin`）もすべて全公開されています。

次回の**第2回**では、この環境の上に **Google Cloud Identity (Free) や GitHub** を ID プロバイダー（IdP）として連携し、**「`/wp-admin` へのアクセスのみを特定ユーザー・特定国・パスキー（生体認証）でピンポイント保護する Access ポリシーの作成」** および **「WordPress の二重ログイン（ID/パスワード入力）の完全撤廃」** を行います！

---

### 次回予告
* **【第2回】/wp-admin だけをピンポイント保護！完全無料のゼロトラストアクセス基盤**
  * GWS（Cloud Identity Free）と GitHub の Identity Provider (IdP) 連携
  * グループ単位での MFA / パキー（Passkey）強制化
  * コンテキスト制御（Geo-blocking: 日本国内限定 / Email ドメイン制限）
  * `mu-plugins` による WordPress 二重ログインの解消（`CF-Access-Authenticated-User-Email` 活用）