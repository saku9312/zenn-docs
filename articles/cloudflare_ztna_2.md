---
title: "【第2回】CloudflareとGWSを使用してゼロトラストを検証してみた　～GWS IdentityFree使用 & IdP連携制御編～"
emoji: "🔑"
type: "tech"
topics: ["cloudflare", "wordpress", "googlecloudidentity", "ztna", "passkey", "GWS"]
published: true
---

前回、CloudflareにDNSを移管し、Cloudflare Tunnelを使ってWordPressを完全に閉域化するところまで構築しました。

ここからは実際にゼロトラストを検証すべく、Cloudflare Accessを使って「/wp-admin」だけをピンポイントで保護する設定を行います。

どのネットワークからアクセスする場合も、**Google Cloud Identityのパスキー（生体認証）＋日本国内IP制限＋特定グループ所属ユーザーのみアクセス許可**という多層防御を検証します

1. **パス指定保護:** サイトトップ（`/`）は全世界へ一般公開し、管理画面（`/wp-admin*`）のみを Cloudflare Access でピンポイント保護。
2. **生体認証（Passkey）強要:** Google Cloud Identity のグループ機能を活用し、パスワード入力を廃止して Touch ID / Windows Hello などの生体認証（パスキー）のみで認証。
3. **コンテキスト制御（ZTNA）:** 日本国内（Geo-blocking）かつ特定の Google グループ所属ユーザーのみアクセスを許可。
4. **二重ログイン完全撤廃:** Cloudflare Access 認証通過後、WordPress のログイン画面を自動バイパスし、ワンタップでダッシュボードを開くヘッダー SSO の導入。

![](https://static.zenn.studio/user-upload/86963ddb750c-20261001.png)

---

## 1. 構成アーキテクチャと認証フロー

今回構築する「ゼロトラスト認証 ＋ ヘッダー SSO」の通信フローは以下のようになります。

```text
[ クライアント (ブラウザ) ]
       │
       │  ① [https://example.com/wp-admin](https://example.com/wp-admin) にアクセス
       ▼
[ Cloudflare Access (Edge) ]
       │
       ├─ ② 未認証の場合 ➔ Google Cloud Identity (IdP) へリダイレクト
       │                        └─ ③ パスキー（指紋/顔認証）で認証実行
       │
       ├─ ④ ポリシー判定 (国: Japan ＆ 所属グループ: testallow ＆ WARPアプリ)
       │
       │  ⑤ 認証成功！ HTTP ヘッダー付与
       │     `CF-Access-Authenticated-User-Email: user@example.com`
       ▼
[ Cloudflare Tunnel (cloudflared) ]
       │
       ▼
[ WordPress (Proxmox LXC) ]
       └─ [ mu-plugins (PHP) ] ── ⑥ ヘッダー検知して即時ログイン ──> [ WP Dashboard ]
```

---

## 2. Google Cloud Identity (Free) のセットアップと認証強化

Cloudflare Access の認証基盤（IdP）として、Google が提供する無料の ID 管理サービス **Google Cloud Identity Free** を使用します。

### Step 1: Cloud Identity Free の準備とグループ作成

1. [Google 管理コンソール](https://admin.google.com/) にログインします。
2. **[ディレクトリ] > [グループ]** を開き、**[グループを作成]** をクリックします。
3. 管理者アクセスを許可するためのグループ（例: `testallow@example.com`）を作成し、対象のユーザーを追加します。

![](https://static.zenn.studio/user-upload/d2a52b9b2b84-20261001.png)

---

### Step 2: MFA の強制とパスワードレス（Passkey）化

パスワード漏洩リスクを物理的にゼロにするため、アカウントに対してパスキー（生体認証）を適用します。

1. **[セキュリティ] > [認証] > [2 段階認証プロセス]** を開きます。
2. 2 段階認証の適用を **「オン」** にし、登録方法としてセキュリティ キー / パスキーを許可します。
3. **[セキュリティ] > [認証] > [パスワードレス]** を開きます。
4. **「ユーザーがパスキーを使用してログインするのを許可する（パスワードの代わりにパスキーを使用）」** を有効化します。

![](https://static.zenn.studio/user-upload/946881e3ca45-20261001.png)
![](https://static.zenn.studio/user-upload/5137efb4cebf-20261001.png)

これにより、日常のログイン時にパスワード入力画面が表示されなくなり、PC やスマホの指紋・顔認証（Touch ID / Face ID / Windows Hello）を掲げるだけでログインできるようになります。

---

## 3. Cloudflare への IdP 統合 (GWS / GitHub)

Cloudflare Zero Trust に Google と GitHub を認証プロバイダーとして登録します。

### Step 1: Google Workspace / Cloud Identity の連携

1. [Cloudflare Zero Trust ダッシュボード](https://one.dash.cloudflare.com/) ➔ **[Settings] > [Authentication]** を開きます。
2. **[Identity providers]** の **[Add new]** をクリックし、**[Google Workspace]** を選択します。

![](https://static.zenn.studio/user-upload/52297fda7102-20261001.png)
![](https://static.zenn.studio/user-upload/8d30d63846ee-20261001.png)

3. Google Cloud Console で OAuth 2.0 クライアント ID とクライアント シークレットを発行し、Cloudflare 側の画面に入力します。

#### 必須スコープとグループ情報の共有
Google グループ判定を Cloudflare で利用する場合は、OAuth 設定時に以下のスコープが含まれていることを確認してください：
* `email`
* `profile`
* `https://www.googleapis.com/auth/admin.directory.group.readonly`

---

### Step 2: GitHub を副 IdP として統合

バックアップ用のログイン手段や外部コラボレーター向けに GitHub 認証も追加します。

1. GitHub の **[Settings] > [Developer settings] > [OAuth Apps]** で新しい OAuth アプリを作成します。
   * **Authorization callback URL:** `https://<your-team-name>.cloudflareaccess.com/cdn-cgi/access/callback`
2. 発行された **Client ID** と **Client Secret** を Cloudflare の **[Authentication] > [GitHub]** に登録します。

![](https://static.zenn.studio/user-upload/ecf3cd39ae4d-20261001.png)
![](https://static.zenn.studio/user-upload/b7cf759fb3a6-20261001.png)

---

## 4. Cloudflare Access アプリケーションの設定（特定パス保護）

ここが ZTNA の核心です。ドメイン全体に鍵をかけるのではなく、**`/wp-admin`（および `wp-login.php`）のみを保護対象**にします。

### Step 1: Self-Hosted Application の作成

1. **[Access] > [Applications]** ➔ **[Add an application]** を選択します。
2. **[Self-Hosted]** を選択します。
3. アプリケーションの基本情報を設定します：

| 項目 | 設定値 | 補足 |
| :--- | :--- | :--- |
| **Application name** | `WordPress Admin Protect` | 任意の識別名 |
| **Session Duration** | `24 hours` | 認証セッションの保持期間 |
| **Domain** | `test-wp.example.com` | 第1回で作成したサブドメイン |
| **Path** | `wp-admin*` | **※最重要：管理画面配下のみを指定** |

![](https://static.zenn.studio/user-upload/84f86e6cfe06-20261001.png)

4. 必要に応じて、`wp-login.php` を保護対象に追加するため、2 つ目の Path として `wp-login.php` も追加登録します。

---

### Step 2: ZTNA ポリシーの構築 (Include / Require / Exclude)

Access ポリシー（Policy）を作成し、アクセスできる人物・条件を多層的に制約します。

* **Action:** `Allow`

#### 1. Include（対象者の指定）
* **Selector:** `Google Groups` ➔ `testallow@example.com`

#### 2. Require（追加条件・コンテキスト制限）
* **Selector:** `Country` ➔ `Japan` （海外 IP からのアクセスを即座にブロック）
* **Selector:** `Warp` ➔ `Warp`（必要に応じて WARP アプリ接続を必須化）

![](https://static.zenn.studio/user-upload/e0839d654923-20261001.png)

---

## 5. WordPress の二重ログインを完全撤廃する（Header SSO）

Cloudflare Access の認証を通過しても、通常は WordPress 自身のログイン画面（ユーザー名とパスワード入力）が再び表示されてしまいます。

Cloudflare Tunnel によってポートが外部から完全に遮断されている安全性を活かし、Cloudflare Access が付与する認証ヘッダー（`CF-Access-Authenticated-User-Email`）を読み取って**自動ログイン（バイパス）させるコード**を仕込みます。

### Step 1: `mu-plugins`（Must-Use Plugins）の作成

WordPress サーバーの `wp-content/mu-plugins/` ディレクトリに PHP スクリプトを作成します。（プラグイン一覧画面から誤って無効化されるのを防ぐため `mu-plugins` を使用します）

```bash
# ディレクトリの作成
mkdir -p /var/www/wordpress/wp-content/mu-plugins/

# スクリプトファイルの作成
nano /var/www/wordpress/wp-content/mu-plugins/cloudflare-sso.php
```

---

### Step 2: ヘッダー自動ログイン用 PHP コード

以下のコードを貼り付けて保存します。

```php
<?php
/*
Plugin Name: Cloudflare Access Header SSO
Description: Cloudflare Access 認証成功時に WP の二重ログインを自動バイパスする
Version: 1.0
*/

if (!defined('ABSPATH')) {
    exit; // 直接アクセスを防止
}

add_action('init', function() {
    // 既に WordPress ログイン済みの場合はスキップ
    if (is_user_logged_in()) {
        return;
    }

    // Cloudflare Access から渡される認証済みメールアドレスの検証
    $cf_email = $_SERVER['HTTP_CF_ACCESS_AUTHENTICATED_USER_EMAIL'] ?? '';

    if (!empty($cf_email)) {
        $email = sanitize_email($cf_email);

        // 【パターン A】Google アドレスと WP のメールアドレスが一致している場合
        $user = get_user_by('email', $email);

        // 【パターン B】一致しない場合のフォールバック（管理者 ID: 1 にマッピングする場合）
        if (!$user) {
            // 必要に応じて特定の WP 管理者ユーザーID を指定可能
            $user = get_user_by('id', 1);
        }

        if ($user) {
            // WordPress ユーザーセッションの発行
            wp_set_current_user($user->ID);
            wp_set_auth_cookie($user->ID, true);
            do_action('wp_login', $user->user_login, $user);

            // ログインページ（wp-login.php）にアクセスしていた場合はダッシュボードへダイレクト転送
            if (strpos($_SERVER['REQUEST_URI'], 'wp-login.php') !== false || strpos($_SERVER['REQUEST_URI'], 'wp-admin') !== false) {
                wp_redirect(admin_url());
                exit;
            }
        }
    }
});
```

---

### 安全性の確認（なぜヘッダー偽装されないのか？）

通常、`$_SERVER['HTTP_CF_ACCESS_AUTHENTICATED_USER_EMAIL']` のような HTTP ヘッダーを信用した認証は、攻撃者が curl 等でヘッダーを偽装してリクエストを送ると破られます。

しかし、本環境では**Cloudflare Tunnel 経由以外のアクセス経路（ポート 80/443）がルーターレベルで存在しない**ため、インターネット上の攻撃者は直接 WordPress サーバーにリクエストを送ることが不可能です。必ず Cloudflare エッジで Access 認証を通過した通信のみが届くため、ヘッダーの偽装攻撃は不成立となります。

---

## 6. 【検証コラム】Free プランで「できなかったこと」と制限の壁

今回、Cloudflare Zero Trust を使い倒す中で、**無料プラン（Free）の制限により設定できなかったエンタープライズ機能**についても検証・特定しました。

### 1. リスクベース認証（User Risk Score）
ポリシーの Selector 内に `User Risk Score` という項目が存在しますが、これは **Enterprise プラン限定機能** です。無料プランでは選択肢に出てこないか、未判定（`Unscored`）となり機能しません。

### 2. 高度な mTLS（端末証明書）による厳密な端末制限
独自のルート CA から発行したクライアント証明書をブラウザに組み込み、証明書が無い端末を完全に拒否する「mTLS 厳密制御」は、Access の高度な設定およびエンタープライズ領域となります。

#### 💡 無料枠での代替案
* **Google 側の「デバイス承認（Device Approval）」機能:** 新規端末からのログイン時に管理者承認を必須化する。
* **Cloudflare WARP 必須化:** 端末に WARP アプリを導入させ、特定の Device Posture（OS バージョンや接続組織）を満たした端末のみ許可する。

---

## 7. 動作検証

すべての設定が完了したら、シークレットウィンドウを開いて動作テストを行います。

1. `https://test-wp.example.com/` にアクセス ➔ **そのまま表示される（OK）**
2. `https://test-wp.example.com/wp-admin` にアクセス ➔ **Cloudflare Access 認証画面（Google / GitHub）が表示される（OK）**
3. Google を選択して認証 ➔ **指紋/顔認証（パスキー）のポップアップが起動（OK）**
4. 認証成功 ➔ **WordPress のログイン画面をスルーして、そのままダッシュボードが開く！（OK）**

![](https://static.zenn.studio/user-upload/d8746e3842cd-20261001.png)
![](https://static.zenn.studio/user-upload/41f79a8981fa-20261001.png)

---

## 8. まとめと次回予告

第2回では、**特定パスへの Zero Trust アクセス制御（国制限・パスキー・グループ権限）** と **ヘッダー SSO による二重ログイン撤廃** を完成させました。

これで、管理画面への攻撃（ブルートフォースや海外からの不正アクセス）をエッジで 100% 撃沈しつつ、自分自身は指紋認証 1 秒で管理画面に入れる極めて快適な環境が整いました。

最終回となる**第3回**では、さらに「遊び心と実用性」を追求します。  
**Cloudflare Workers ＋ KV ＋ LINE Messaging API** を組み合わせて、`/wp-admin` にアクセスがあった際に自分の LINE へ通知を飛ばし、**「LINE で許可ボタンを押した後の 10 分間だけアクセスを開放する JIT（Just-In-Time）承認システム」** を自作します！

---

### 次回予告
* **【第3回】LINEで「ぽちっ」と10分間限定開通！Cloudflare Workersで作るChatOps認証**
  * Just-In-Time (JIT) アクセス制御のコンセプト
  * LINE Messaging API (Webhook) の環境構築
  * Cloudflare KV（Key-Value Store）による 600 秒（10分）TTL フラグ管理
  * Workers（JavaScript）のコード実装と動作デモ