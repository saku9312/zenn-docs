---
title: "完全無料で構築するゼロトラスト環境：Cloudflare Gatewayで行うTLS復号・QUIC制御・疑似CASBとポスチャーによるデスクトップアプリ制御"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "zerotrust", "casb", "network", "security"]
published: false
---

## 1. はじめに

これまでCloudflare Oneを使って、特定サーバーへのアクセスを他サービスとのアカウント連携で制御するZTNAを実装したり、OktaとのSAML連携など、テストを重ねてきました。

今回は、その内容に加えて実施した「HTTPS通信の傍受」、「ヘッダー情報からのテナント制御」、「デスクトップアプリの起動状況によるポスチャーチェック」をまとめてみます。

ここからほぼAIで書いていますが、テスト環境で実装しているので悪しからず。



さて、ZTNAやSSOでのアカウントおよび接続制限をする中で、より深い制御が必要になってきています。

そんな中、求められるのが**「ゼロトラストアーキテクチャ」**や、シャドーIT（会社の許可を得ていない私用クラウドや個人アカウントの利用）を防ぐ**「CASB（Cloud Access Security Broker）」**、そしてセキュアWebゲートウェイ（SWG）です。

しかし、市販のエンタープライズ向けCASB/SSE製品や次世代ファイアウォール（NGFW）を導入しようとすると、莫大な初期費用やユーザー単位の月額高額ライセンス、専用ハードウェアの保守費用が発生します。特に中小規模の組織やスタートアップ、コストを最小限に抑えたい情シス・セキュリティ担当者にとって、導入のハードルは極めて高いのが現実です。

本記事では、**Cloudflare Zero Trust の無料枠（最大50ユーザーまで無料）**をフル活用し、完全無料で以下のエンタープライズ級セキュリティ制御を実現する具体的な手順と技術的背景を徹底解説します。

- **クラウドプロキシ（SWG）によるセキュアなインターネットアクセス**
- **自社専用ルートCA証明書の自動配布と完全なTLS復号（HTTPSインスペクション）**
- **次世代通信規格「QUIC（UDP 443）」のバイパス阻止と完全制御**
- **HTTPヘッダーインジェクションによる「疑似CASB（Google Workspace等のテナント制限）」**
- **WARPの「デバイスポスチャーチェック（Application Check）」を活用した未許可デスクトップアプリの稼働検知とアクセス制御**

自前でSquidなどのプロキシサーバーを運用する手間から解放され、クラウドネイティブかつ高可用性なセキュリティ基盤を構築しましょう。

---

## 2. 従来型オンプレミス・自前プロキシ（Squid等）との比較

Web通信の可視化や制御を行う際、従来はオンプレミスにSquidやUTMを設置し、端末のプロキシ設定（PACファイルやWPAD）で通信を集中させるアプローチが一般的でした。

しかし、現代の多様な働き方において、この方式には致命的な限界が存在します。

| 比較項目 | 従来型（Squid / オンプレミスプロキシ） | Cloudflare Zero Trust (WARP + Gateway) |
| :--- | :--- | :--- |
| **初期・運用コスト** | サーバー機材費、OS保守、電気代、冗長化費用 | **完全無料**（50ユーザーまで） |
| **回線のボトルネック** | 全社通信が社内回線に集中（輻輳・遅延の原因） | 世界300都市以上のCloudflareエッジで分散処理 |
| **端末のロケーション** | 社外端末はVPN接続が必須（VPNの負荷増大） | 自宅やカフェ、モバイル回線でも自動保護 |
| **証明書の配布・管理** | 手動配布、Active Directory GPO等での配布が必要 | **WARPクライアントによる完全自動インストール** |
| **プロトコル制御** | QUIC等の新プロトコル対応が困難・遅い | L4（Network）〜L7（HTTP）を一元的に高速制御 |
| **端末健全性連携** | 単体では端末状態（EDRや常駐アプリ）と連動不可 | **Device Posture機能で端末状態と即時連動** |

Squidなどの自前プロキシを管理する場合、定期的な脆弱性対応（パッチ適用）、負荷に応じたスケールアウト、SSLバンプ（TLS復号）のためのハードウェアリソース確保など、多大な運用コストが発生します。

Cloudflare Zero Trustへ移行することで、インフラ保守から完全に解放され、純粋なセキュリティポリシーの設計と運用に集中できます。

---

## 3. 全体アーキテクチャと制御フロー

本構成の全体像は以下の通りです。

~~~text
[ エンドポイント端末 (Windows / macOS / Linux) ]
  │
  ├─ ① Cloudflare WARP Client が常駐
  │    ├─ クラウドから自社専用ルートCA証明書を自動同期・インストール
  │    ├─ 常駐プロセス（デスクトップアプリ）やOS状態を定期ポスチャーチェック
  │    └─ 全アウトバウンド通信を暗号化トンネルでCloudflare Edgeへ転送
  │
  ▼ (WireGuardベースのセキュアトンネル)
[ Cloudflare Global Network (Gateway) ]
  │
  ├─ ② Network Policy (L4)
  │    └─ QUIC (UDP 443) を強制ブロック ➔ ブラウザを標準HTTPS (TCP 443) にフォールバック
  │
  ├─ ③ HTTP Policy (L7) - TLS Decryption
  │    ├─ 自社専用CAで通信を動的に復号・インスペクション
  │    └─ 【疑似CASB】Google Workspace宛て通信にテナント制限ヘッダーを注入
  │         (X-GoogApps-Allowed-Domains: your-company.com)
  │
  ├─ ④ Access / Device Posture Policy
  │    └─ 【端末アプリ制御】未許可デスクトップアプリ稼働端末からの社内アクセスを遮断
  │
  ▼
[ 外部インターネット / 各種SaaS (Google Workspace, M365 etc.) ]
  └─ SaaS側がヘッダーを検証し、個人アカウントでのログインを拒否
~~~

すべての通信の起点は端末上のWARPクライアントであり、CloudflareのエッジサーバーでL4（ネットワーク層）およびL7（アプリケーション層）のポリシーが順次適用されます。

---

## 4. ルートCA証明書の生成と展開（TLS復号の土台）

HTTPS通信の中身（HTTPヘッダー、URLパス、POSTデータなど）を検査・改ざん・ブロックするためには、端末とCloudflare Gatewayの間で**TLS復号（HTTPSインスペクション / SSLバンプ）**を行う必要があります。

### 4.1 なぜ「中間証明機関」ではダメで、「信頼されたルート証明機関」なのか？

TLS復号の仕組みは、一種の「制御された中間者（Man-in-the-Middle）」動作です。端末が `google.com` にアクセスしようとすると、Cloudflare Gatewayがその場で `google.com` の偽造証明書を動的に発行し、端末に返却します。

このとき、ブラウザやOSは提示された証明書が信頼できるものか、証明書チェーン（信頼の輪）を最上位まで検証します。

~~~text
【ブラウザが検証するチェーンの流れ】
1. Webサイトの証明書 (Cloudflareが動的に生成した google.com)
   └── 親は誰か？ ➔ Cloudflare Gateway CA

2. Cloudflare Gateway CA の証明書 (今回配布するもの)
   └── 親は誰か？ ➔ 自分自身（自己署名）
~~~

- **「信頼されたルート証明機関」に配置した場合**:
  OSは「この証明書自体が、システム管理者が明示的に許可した最上位の親（トラストアンカー）である」と認識し、チェーンの検証を正常終了して通信を許可します。
- **誤って「中間証明機関」に配置した場合**:
  OSは「これは中間地点だ。では、この証明書に署名したさらに上の大元のルート証明書はどこにある？」と探索を続けます。しかし、自己署名であるCloudflare Gateway CAより上の親は存在しないため、**「信頼できない発行元」「チェーンが不完全」**と判定され、ブラウザに `NET::ERR_CERT_AUTHORITY_INVALID` などの致命的なセキュリティ警告が表示されます。

そのため、必ず**「信頼されたルート証明機関（Local Machine \ Trusted Root Certification Authorities）」**へ登録しなければなりません。

### 4.2 テナント専用CAの生成と有効化

近年のCloudflare Zero Trustでは、共有のルート証明書ではなく、**組織（テナント）ごとに固有のルートCAを生成して運用する仕様**になっています。

1. Cloudflare Zero Trust ダッシュボードにログインします。
2. 左側メニューから **[Traffic policies] > [Traffic settings]**（または **[Settings] > [Resources]**）を開きます。
3. **[Certificates]** タブを選択し、**[Generate certificate]** をクリックします。
4. 有効期間（推奨は5年）を選択し、生成を実行します。
5. 生成直後はステータスが `Inactive` になっています。証明書の右側メニューから **[Activate]** を選択します（ステータスが `Available` に遷移）。
6. 基本情報欄から **[Confirm and turn on certificate]** を実行し、ステータスを `Available and In-Use` に切り替えます。これでGatewayでの復号準備が完了します。

:::message
**手動インストールは原則不要です**
端末にCloudflare One（WARP）クライアントがインストールされており、対象のZero Trust組織にサインインしている場合、証明書を `Activate` にした時点で**WARPクライアントがOSの証明書ストアへ自動的にバックグラウンド登録**を行います。
:::

### 4.3 Windows端末での自動インストール確認

Windows環境において、証明書が正常に同期されたかを確認するには、管理者権限のPowerShellまたは「ファイル名を指定して実行」から確認します。

~~~powershell
# ローカルコンピューターのルート証明機関ストアを確認
certlm.msc
~~~

**[信頼されたルート証明機関] > [証明書]** の配下に、**`Gateway CA - Cloudflare Managed G1`**（または自社テナント名が付与された証明書）が存在していれば、正常に同期が完了しています。

手動インポートが必要な検証環境などの場合は、ダッシュボードから `.crt` または `.pem` をダウンロードし、以下のPowerShellコマンドでローカルマシンストアへ登録可能です。

~~~powershell
# 手動インポート用コマンド（管理者権限PowerShell）
Import-Certificate -FilePath "C:\path\to\Cloudflare_CA.crt" -CertStoreLocation Cert:\LocalMachine\Root
~~~

---

## 5. 最大の落とし穴「QUIC（UDP 443）」の完全遮断

TLS復号とプロキシ環境を構築するエンジニアが最も頻繁に遭遇し、時間を浪費する落とし穴が**「QUICプロトコル（HTTP/3）」の存在**です。

### 5.1 QUICとは何か？ なぜUDP 443なのか？

QUICは、GoogleがWeb通信の高速化を目的に開発し、2021年5月にIETFによって正式に国際標準化（**RFC 9000**）されたプロトコルです。現在のHTTP/3のベース技術となっています。

長年、Web通信の標準であったTCPは、接続確立時の3-way handshakeやTLSハンドシェイクにより、データ送受信が開始されるまでに複数回の往復通信（RTT）が必要でした。
Googleは、世界中のネットワーク機器やOSカーネルに手を加えることなく高速な暗号化通信を実現するため、**「OS側が素通ししてくれるUDPの上に、自前で信頼性制御とTLS 1.3を一体化して実装する」**というアプローチを取りました。

さらに、オフィスのファイアウォールやホテルの公衆Wi-Fiで不審なポートとして遮断されることを避けるため、既存のHTTPSと同じポート番号である **「UDP 443番」** を使用する仕様にしました。

### 5.2 なぜQUICがセキュリティの「抜け穴」になるのか

ChromeやEdgeなどのモダンブラウザは、Google Workspace（Gmail, Drive等）やYouTube、Cloudflare上のサイトにアクセスする際、TCP 443ではなく**UDP 443（QUIC）での接続を最優先で試行**します。

しかし、一般的なプロキシ製品やCloudflare Gatewayの標準的なHTTPインスペクションは、TCPストリームを中継・終端するように設計されています。UDP 443の通信がファイアウォールを素通りしてしまうと、**ブラウザとSaaSがプロキシを迂回してダイレクトに暗号化通信を確立してしまう**ため、後述するHTTPヘッダー付与やURLフィルタリングが一切機能しなくなります。

### 5.3 Network Policy による QUIC の遮断設定

これを防ぐためのエンタープライズの鉄則は、**「ネットワーク境界で UDP 443 を確実にブロックする」**ことです。ブラウザはQUICでの疎通に失敗すると、コンマ数ミリ秒のレベルで自動的に従来のTCP 443（標準HTTPS）へフォールバックします。

1. ダッシュボードから **[Traffic policies] > [Network policies]** を開きます。
2. **[Add a policy]** をクリックします。
3. ルールを以下のように定義します。

| 設定項目 | 入力値 / 選択肢 | 補足 |
| :--- | :--- | :--- |
| **Policy Name** | `Block-QUIC-UDP443` | 任意の分かりやすい名前 |
| **Selector** | `Destination Port` | 宛先ポート |
| **Operator** | `==` (is) | 完全一致 |
| **Value** | `443` | ポート番号 |
| *(AND条件追加)* | **[+ Add condition]** をクリック | 複合条件にする |
| **Selector** | `Protocol` | 通信プロトコル |
| **Operator** | `==` (is) | 完全一致 |
| **Value** | `UDP` | UDP通信のみを対象化 |
| **Action** | **`Block`** | パケットを破棄 |

4. **[Create policy]** をクリックして保存します。

これで、ブラウザは強制的にTCP 443での通信を強いられ、Cloudflare GatewayのHTTPインスペクションパイプラインを確実に通過するようになります。

---

## 6. HTTP復号の有効化とセレクタの仕様（URL vs Domain vs Host）

続いて、L7（HTTP層）での復号とポリシー設定を行います。

### 6.1 TLS Decryptionの有効化

1. **[Traffic policies] > [Traffic settings]** を開きます。
2. **[TLS decryption]**（または Inspect HTTPS requests with TLS decryption）のトグルを **ON** にします。
3. 必要に応じて、銀行系サイトや公的機関など、法律やプライバシー規約により復号が禁止されている通信をバイパスする例外設定（Do Not Inspect）を適用します。

### 6.2 ポリシー作成時の落とし穴：`relative URL without a base`

HTTPポリシーを作成する際、初心者が最もつまずきやすいのがセレクタの選定です。
例えば、Yahooなどの特定ドメインをブロックしようとして、以下のように入力するとエラーになります。

- **Selector**: `URL`
- **Operator**: `contains`
- **Value**: `yahoo.co.jp`
- **エラー表示**: `relative URL without a base`

`URL` セレクタは、`https://example.com/path/index.html` のような「スキーム（http/https）から始まる完全な絶対URL」を要求します。ドメイン名単体を入れると、ベースURLが存在しない相対パスと判定されて構文エラーになります。また、Cloudflareのルール入力欄では `*` による単純なワイルドカード指定はサポートされておらず、パターンマッチには正規表現（`matches regex`）を用いる必要があります。

### 6.3 セレクタの正しい使い分け

用途に応じて、以下のセレクタを使い分けるのがベストプラクティスです。

1. **ドメイン全体およびサブドメインをまとめて制御したい場合【最も推奨】**
   - **Selector**: **`Domain`**
   - **Operator**: `in` または `is`
   - **Value**: `yahoo.co.jp`
   - *解説*: `Domain` セレクタを指定すると、`yahoo.co.jp` 本身だけでなく、`news.yahoo.co.jp` などの配下サブドメインすべてに自動でマッチします。
2. **ホスト名単位で完全一致させたい場合**
   - **Selector**: **`Host`**
   - **Operator**: `is`
   - **Value**: `yahoo.co.jp`
3. **URLのパスやクエリ文字列まで細かく判定したい場合**
   - **Selector**: **`URL`**
   - **Operator**: `matches regex`
   - **Value**: `^https?://.*yahoo\.co\.jp/.*$`

---

## 7. 疑似CASBの構築（テナント制限による個人アカウント排除）

TLS復号とQUIC制御が完了したことで、本丸である**疑似CASB（テナント制限）**の実装が可能になります。

### 7.1 テナント制限（Tenant Restriction）の動作原理

「自社契約のSaaSは使わせたいが、個人のアカウントでのログインは完全に防ぎたい」という課題に対し、Cloudflare Gatewayは**HTTPヘッダーインジェクション**を用いたアプローチを取ります。

ここで極めて重要なアーキテクチャ上の理解は、**「通信を遮断（拒否）するのはCloudflareではなく、SaaS（GoogleやMicrosoft）側である」**という点です。

~~~text
[ 端末 (ブラウザ) ]
  │ ① @gmail.com (個人アカウント) でログインを試行
  ▼
[ Cloudflare Gateway ]
  │ ② ポリシー: Action = "Allow"
  │    通信は通すが、リクエストヘッダーに以下を強制付与してGoogleへ転送する
  │    Header: X-GoogApps-Allowed-Domains
  │    Value : your-company.com (自社ドメイン)
  ▼
[ Google 認証サーバー (accounts.google.com) ]
  │ ③ Google側の処理:
  │    「リクエストに『your-company.com のみ許可』と指定されているが、
  │      ユーザーが入力したアカウントは @gmail.com である」
  ▼
【 Google側でアクセス拒否画面を表示 】
「このアカウントは、このネットワークからはアクセスが許可されていません」
~~~

Cloudflare側のアクションが `Block` ではなく **`Allow`** である理由はここにあります。Cloudflareはあくまで「このリクエストは我が社の管理下にある」という名札を貼り付けて通すだけであり、名札を見て門前払いを食らわせるのは宛先のSaaS側です。

### 7.2 Google Workspace テナント制限の実装手順

1. ダッシュボードから **[Traffic policies] > [HTTP policies]** を開きます。
2. **[Add a policy]** をクリックします。
3. 以下の内容で設定を行います。

#### ① ポリシー基本情報
- **Policy Name**: `Tenant-Control-GoogleWorkspace`

#### ② 評価条件（Build an expression）
:::message alert
`Application: Google Workspace` をセレクタに選ぶと、ログイン基盤である `accounts.google.com` への通信が漏れる場合があります。確実に捕捉するため、セレクタには **`Domain`** を使用します。
:::

- **Selector**: **`Domain`**
- **Operator**: `in`
- **Value**: `google.com`

#### ③ アクションとヘッダー設定
- **Action**: **`Allow`**
- 画面下部の **[HTTP request headers]** を展開し、**[Add header]** をクリックします。
- **Header name**: `X-GoogApps-Allowed-Domains`
- **Header value**: `your-company.com`（※自社のGoogle Workspaceドメイン名。`@` は含めない。複数ドメインを許可する場合はカンマ区切りで入力）

4. **[Create policy]** をクリックして保存します。

### 7.3 動作確認のチェックポイント

設定が反映されたら、必ず**ブラウザのシークレットウィンドウ（InPrivateウィンドウ）**を新規で立ち上げてテストします。既存の通常タブでは以前のログインCookieやセッションキャッシュが残り、正しい検証ができません。

1. シークレットウィンドウで `https://accounts.google.com` にアクセスします。
2. アドレスバーの鍵マークをクリックし、証明書の発行元が **「Cloudflare」** になっていることを確認（TLS復号の確認）。
3. 自社の組織アカウント（`user@your-company.com`）を入力 ➔ **正常にログインできることを確認**。
4. 一度ログアウトし、私用のアカウント（`user@gmail.com`）を入力 ➔ **Googleの認証画面で「アクセスが制限されています」等のエラー画面が表示され、ログインが遮断されることを確認**。

これで、完全無料の疑似CASBが完成しました。

### 7.4 他の主要SaaSへの横展開

同様のHTTPヘッダーインジェクションを用いることで、Google以外の主要SaaSでも個人アカウントや野良テナントの利用を完全に封じ込めることが可能です。

~~~markdown
| 対象SaaS | 制御したい宛先ドメイン | 付与するHTTPヘッダー名 | 設定値（Header Value） |
| :--- | :--- | :--- | :--- |
| **Microsoft 365** | `login.microsoftonline.com` | `Restrict-Access-To-Tenants` | 自社テナントID（GUID）またはドメイン |
| | | `Restrict-Access-Context` | 自社ディレクトリID |
| **Slack** | `slack.com` | `X-Slack-Allowed-Workspaces` | 自社ワークスペースID (Tから始まるID) |
| **Dropbox** | `dropbox.com` | `X-Dropbox-allowed-Team-Ids` | 自社チームID |
| **ChatGPT (OpenAI)**| `chatgpt.com` | `Chatgpt-Allowed-Workspace-Id`| 自社ワークスペースID |
~~~

---

## 8. Webだけではない！ポスチャーチェック（Application Check）によるデスクトップアプリ制御

WebブラウザからのSaaSアクセスをCASBで完璧に保護しても、それだけではセキュリティ対策として片手落ちです。多くのクラウドストレージやチャットツールには、**「ローカルにインストールして動作する専用デスクトップアプリケーション」**が存在するからです。

### 8.1 デスクトップアプリが引き起こすセキュリティホール

例えば、従業員がPCローカルに個人用のDropboxクライアント（`Dropbox.exe`）や未許可のメッセンジャー（LINE、Discord等）をインストールした場合、以下のようなリスクが生じます。

1. **同期フォルダによる自動データ流出**:
   社内ファイルをローカルのDropbox同期フォルダにドラッグ＆ドロップするだけで、バックグラウンド通信により個人ストレージへファイルが自動アップロードされてしまう。
2. **ブラウザベースCASBの迂回**:
   デスクトップアプリ独自の実装（SSLピンニングや独自プロトコル）により、GatewayでのTLS復号を通過しにくく、Web向けのテナント制限ヘッダーが正しく解釈されない場合がある。

この課題を解決するのが、Cloudflare WARPが標準で備えている**「デバイスポスチャーチェック（Device Posture Check - Application check）」**です。

### 8.2 Application check（プロセス・ファイル監視）の仕組み

WARPクライアントは、単なるVPN/プロキシトンネルを提供するだけでなく、**OS内で稼働しているプロセスや指定パスのファイル存在をローカルレベルで常時監視（ポスチャーチェック）**する機能を備えています。

~~~text
[ 端末 (OS) ]
  │
  ├─ WARP Client が常時バックグラウンドで監視
  │    ├─ 指定されたプロセスの実行状態（Running / Not Running）
  │    ├─ 実行ファイルの存在パス（File Path）
  │    └─ 実行ファイルのSHA256ハッシュ値（改ざん検知）
  │
  ▼ 状態レポート（安全 / 危険）をCloudflare Edgeへ常時送信
[ Cloudflare Access / Gateway ]
  │
  ▼ ポリシー評価:
     「未許可アプリ（例: Dropbox.exe）が稼働中の端末は、
       社内アプリやWebアクセスを即座にブロックする」
~~~

このポスチャー判定結果をCloudflare AccessのポリシーやGatewayポリシーの条件（Selector）として組み込むことで、**「危険なデスクトップアプリが起動している間は、業務システムへの一切の通信を遮断する」**という動的なアクセス制御が実現します。

### 8.3 具体的な設定手順：未許可デスクトップアプリの排除

ここでは、「個人用Dropboxのデスクトップクライアントが実行されている端末を遮断する」ケースを例に設定手順を解説します。

#### 手順①：ポスチャーチェック（Application Check）の定義

1. ダッシュボードから **[Settings] > [WARP Client]**（または **[Team and resources] > [Devices] > [Posture checks]**）を開きます。
2. **[Add a check]** をクリックし、リストから **[Application]** を選択します。
3. 監視したいアプリケーションの情報を定義します。

| 設定項目 | 設定値の例（Windowsの場合） | 補足 |
| :--- | :--- | :--- |
| **Name** | `Check-Dropbox-Running` | ポスチャーの識別名 |
| **Operating system** | `Windows` | 対象OS（macOS / Linuxも選択可） |
| **Path** | `C:\Program Files (x86)\Dropbox\Client\Dropbox.exe` | 実行ファイルの絶対パス |
| **Application status**| **`Running`** | プロセスが実行中かどうかをチェック |

※必要に応じて、実行ファイルの改ざんを防ぐための「SHA256 Checksum」や、ユーザープロファイル配下のパス（環境変数 `%APPDATA%` 等を含むパス）を指定することも可能です。

4. **[Save]** をクリックしてポスチャーチェックを登録します。

#### 手順②：Accessポリシーへのバインド

定義したポスチャーチェックを、社内システムやCloudflare Accessで保護されたアプリケーションのルールに組み込みます。

1. **[Access controls] > [Applications]** を開き、保護対象の社内アプリケーション（またはGatewayポリシー）を選択します。
2. **[Policies]** タブで、ルールの設定を開きます。
3. ルールの条件に以下を追加します。
   - **Rule Action**: `Block`（または `Require` による否定判定）
   - **Selector**: **`Passed Device Posture Check`**
   - **Value**: 先ほど作成した `Check-Dropbox-Running`

この設定により、端末側で `Dropbox.exe` のプロセスが立ち上がった瞬間、WARPクライアントがそれを検知してエッジに通知し、**社内システムやインターネットアクセスが即時にロック（遮断）**されます。従業員がアプリを終了（プロセス停止）させるまで通信は再開されません。

### 8.4 必須セキュリティアプリ（EDR・Sysmon）の稼働強制への応用

このApplication checkは、攻撃的なシャドーITの排除だけでなく、**「防御ツールの常時稼働強制」**というゼロトラストの原則にも活用できます。

- **Microsoft Defender / EDRの常時監視**:
  `SenseIR.exe`（Defender for Endpoint）やサードパーティEDRが停止した端末を検知して遮断。
- **Sysmon（操作ログ収集）の停止検知**:
  管理者権限で不正にログ収集エージェントが停止された場合、その端末からのネットワーク接続を即座に隔離。

このように、**「L7のHTTPインスペクション（Web・SaaS制御）」**と**「端末側のApplication Posture（デスクトップアプリ制御）」**を組み合わせることで、WebブラウザとOSローカルプロセスの両面から強固なセキュリティ境界を築くことができます。

---

## 9. 運用上の注意点とトラブルシューティング

実際に運用を開始した際によく直面する疑問やトラブルの解決策をまとめました。

### 9.1 ログ仕様の注意点：拒否ログはCloudflare側には残らない

情シス担当者から最も多く受ける相談が、「個人アカウントでログインしようとして弾かれた従業員のメールアドレスを、CloudflareのGatewayログから一覧抽出したい」という要望です。

しかし、**Cloudflare Gatewayのログには、拒否された個人メールアドレス（`xxx@gmail.com`）は記録されません**。

- **理由**:
  前述の通り、ブロックを実行したのはGoogleやMicrosoft側のサーバーであり、Cloudflare側はヘッダーを付与して正常に「Allow（転送）」しただけで通信を終えているためです。CloudflareのHTTPログに記録される `Identity Email` は、あくまで**端末のWARPにログインしている社内ユーザーのID**です。
- **監査方法**:
  「誰が私用アカウントでログインを試みたか」の証跡を確認したい場合は、**Google Workspace管理コンソール（監査ログ > ログインログ）**やMicrosoft Entra IDのサインインログ側を確認する必要があります。SaaS側のログには「テナント制限ポリシーに違反したためブロックされた」記録が確実に残ります。

### 9.2 SSLピンニング（Certificate Pinning）アプリへの対処

一部のデスクトップアプリケーション（Slack、Zoom、Docker、開発系CLIツール、金融系専用ソフトなど）は、セキュリティを高めるために「OSの証明書ストアを無視し、アプリ内部にハードコードされた証明書しか信用しない」**SSLピンニング**を採用しています。

これらのアプリは、Cloudflare GatewayがTLS復号のために証明書を差し替えた瞬間、中間者攻撃とみなして通信エラー（ネットワーク接続不能）を引き起こします。

- **対処法**:
  HTTPポリシーの最上位に **「Do Not Inspect（復号バイパス）」** ルールを配置します。
  - **Selector**: `Application` is `Zoom` / `Slack`
  - **Action**: **`Do Not Inspect`**
  業務上安全であることが分かっている通信のみピンポイントで復号対象から除外することで、業務の継続性を担保します。

---

## 10. まとめ

本記事で構築した環境を振り返ってみましょう。

1. **自社専用ルートCA証明書**を生成し、WARPクライアントを通じて端末へ自動展開した。
2. **Network PolicyでUDP 443（QUIC）を遮断**し、ブラウザの通信を確実にプロキシのインスペクション対象（TCP 443）へ誘導した。
3. **HTTP PolicyでTLS復号**を有効化し、`Domain` セレクタを用いてGoogle等のWebトラフィックを捕捉した。
4. **`X-GoogApps-Allowed-Domains` ヘッダーを注入**することで、完全無料で私用Gmailへのアクセスを遮断する「疑似CASB（テナント制限）」を実現した。
5. **WARPの「Application Posture Check」**を組み合わせ、Webブラウザのみならずローカルで動作する未許可デスクトップアプリ（Dropbox.exe等）の起動も検知・遮断する二重の防御壁を構築した。

市販の高額なCASBやEDR連携ソリューションを契約しなくても、**Cloudflare Zero Trustの無料プランが提供している基本機能（Gateway + WARP + Posture）を正しく組み合わせるだけ**で、ここまで高度なゼロトラスト・セキュリティ基盤を組み上げることができます。

情シスリソースやセキュリティ予算が限られている組織こそ、こうしたクラウドネイティブなアプローチを武器に、低コストかつ堅牢な環境作りを推進してみてはいかがでしょうか。