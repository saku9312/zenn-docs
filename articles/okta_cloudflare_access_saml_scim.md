---
title: "OktaとCloudflare AccessをSAML+SCIM連携してゼロトラスト環境(ZTNA)を構築する"
emoji: "☁️"
type: "tech"
topics: ["Okta", "Cloudflare", "ZeroTrust", "SAML", "SCIM"]
published: true
---

## はじめに

企業内のWebアプリケーションやプライベートなリソース（Cloud Run上で動く社内ツールなど）を保護するため、VPNに代わるアプローチとして**ZTNA（Zero Trust Network Access）**の導入が進んでいます。

この記事では、IDプロバイダ（IdP）として**Okta**、アクセス制御を担うサービスプロバイダ（SP）として**Cloudflare Zero Trust (Access)**を使用し、両者を**SAML**による認証と**SCIM**によるユーザー・グループの自動プロビジョニングで連携させる手順を詳細に解説します。

公式ドキュメント通りに進めても陥りやすい**「数々の罠（ハマりポイント）」**についても、実戦ベースのトラブルシューティングとして後半でまとめています。

---

## 構成アーキテクチャと目的

今回の構成の目的は以下の通りです。

1. **アクセス制御の中央集権化**: 社内向けWebアプリへのアクセス経路の前段にCloudflare Accessを配置する。
2. **強固な認証**: ユーザーはCloudflare Accessの画面でOktaへリダイレクトされ、Okta側で多要素認証（MFA）などを含めた厳格な認証を行う。
3. **自動プロビジョニング（SCIM）**: Okta上のグループ情報をCloudflareに同期し、「Oktaの特定のグループに所属しているユーザーのみアクセスを許可する」というポリシーを自動適用する。

### 認証フロー
1. ユーザーが保護された社内サイト（例: `https://internal.example.com`）にアクセス。
2. Cloudflare Accessがリクエストをインターセプトし、未認証ならOktaのログイン画面へ転送（SAML Request）。
3. ユーザーがOktaで認証を完了すると、Cloudflareへ戻される（SAML Response）。
4. Cloudflareがポリシー（Oktaのグループ所属など）を評価し、許可されればサイトを表示。

---

## 前提条件

- Oktaの管理者権限（Admin Consoleへのアクセス）
- Cloudflare Zero Trustの管理者権限
- Cloudflare Zero Trustの「Team domain」が決定していること（`https://<あなたのチーム名>.cloudflareaccess.com`）

---

## ステップ1：Okta側でSAMLアプリを作成する

まずはOkta上にCloudflare Access用のSAMLアプリケーションを作成し、連携に必要なパラメータを取得します。

### 1-1. アプリケーションの作成
1. Okta Admin Consoleで **[Applications] > [Applications]** を開き、**[Create App Integration]** をクリック。
2. **Sign-on method** で **[SAML 2.0]** を選択してNext。
3. **General Settings** の App name に `Cloudflare Zero Trust` 等と入力してNext。

### 1-2. SAML設定の入力
**SAML Settings** に以下の通り入力します。`<あなたのチーム名>` の部分はご自身の環境に合わせて変更してください。

- **Single sign-on URL**: `https://<あなたのチーム名>.cloudflareaccess.com/cdn-cgi/access/saml`
- **Audience URI (SP Entity ID)**: `https://<あなたのチーム名>.cloudflareaccess.com/cdn-cgi/access/saml`
- **Name ID format**: `EmailAddress`
- **Application username**: `Email`

### 1-3. 属性（Attributes）のマッピング
Cloudflare側でユーザー情報（メールアドレスなど）を正しく認識させるため、**Attribute Statements (Optional)** を必ず追加します。

| Name | Name format | Value |
| :--- | :--- | :--- |
| `email` | Unspecified | `user.email` |
| `firstName` | Unspecified | `user.firstName` |
| `lastName` | Unspecified | `user.lastName` |

設定後、Nextをクリックし、次の画面で「I'm an Okta customer adding an internal app」を選択して **[Finish]** をクリックします。

### 1-4. SAMLメタデータの取得
アプリ作成完了後、**[Sign On]** タブを開き、**[View SAML setup instructions]** をクリックします。以下の3つの情報を手元にコピーしておきます。

- Identity Provider Single Sign-On URL
- Identity Provider Issuer
- X.509 Certificate（`-----BEGIN CERTIFICATE-----` から始まるテキスト）

---

## ステップ2：Cloudflare側でOktaをIdPとして登録する

取得したOktaの情報をCloudflare Zero Trustに教え込みます。

*(※CloudflareのUIは頻繁に変更されます。以前は `Settings > Authentication` にありましたが、現在は `Integrations` に移動しています)*

1. Cloudflare Zero Trust ダッシュボード（`one.dash.cloudflare.com`）を開く。
2. 左メニューから **[Integrations] > [Identity providers]** を開く。
3. **[Add new identity provider]** をクリックし、**[SAML]** を選択。
4. 以下の通り入力します：
   - **Name**: `Okta` （ログイン画面のボタンに表示される任意の名前）
   - **Single Sign-On URL**: ステップ1で取得したURL
   - **IdP Entity ID**: ステップ1で取得したIssuer
   - **Signing certificate**: ステップ1で取得したX.509 Certificateを貼り付け
   - **SAML Attributes**: 
     - Attribute name: `email`
     - Attribute name in SAML assertion: `email`
5. **[Save]** をクリックします。

保存後、一覧の右側にある **[Test]** ボタンを押し、Oktaのログイン画面を経て「Your connection works!」と表示されればSAMLの基本連携は成功です。（※テスト前にOkta側で自分自身をアプリにAssignしておくことをお忘れなく）

---

## ステップ3：SCIMによるプロビジョニング設定

ZTNAのアクセス制御をスケーラブルに運用するためには、「特定のOktaグループに所属しているか」で判定するのがベストプラクティスです。そのためにSCIM機能を使って、OktaからCloudflareへユーザーとグループ情報をPushします。

### 3-1. CloudflareでSCIMエンドポイントを発行
1. Cloudflareの **[Integrations] > [Identity providers]** で、作成したOktaの設定を開く（Edit）。
2. 下部にある **SCIM Provisioning** の設定を探し、有効化（Generate）します。
3. **SCIM Endpoint Base URL** と **SCIM Secret (Bearer Token)** が発行されるのでコピーします。*(※Secretは一度しか表示されないので注意)*

### 3-2. OktaでSCIMを有効化する
1. Okta側のCloudflareアプリで **[General]** タブを開き、App Settingsの Edit をクリック。
2. **Provisioning** の項目で **SCIM** を選択し保存。
3. 新たに出現した **[Provisioning]** タブを開く。

### 3-3. SCIM APIの接続設定
**[Provisioning] > [Integration] > [Edit]** を開き、Cloudflareの情報を入力します。

- **SCIM connector base URL**: CloudflareのEndpoint URL
- **Unique identifier field for users**: `userName` （※ `email` と間違えやすいので注意！）
- **Supported provisioning actions**: `Push New Users`, `Push Profile Updates`, `Push Groups` にチェック。
- **Authentication Mode**: `HTTP Header` を選択。（※デフォルトの `Basic Auth` ではない）
- **Authorization**: CloudflareのSCIM Secretをそのまま貼り付け。

入力後、**[Test Connector Configuration]** をクリックし、成功したら保存します。

### 3-4. To App の設定
**[Provisioning] > [To App] > [Edit]** を開き、以下の項目を **Enable** にします。
- Create Users
- Update User Attributes
- Deactivate Users
*(※Sync Password は不要なので Disabled)*

### 3-5. グループのPushとユーザーの割り当て
1. **[Push Groups]** タブを開き、Cloudflare側でアクセスポリシーに使用したいグループ（例: `GWS-SSO-Users` や `Engineering`）をPushして `Active` にします。
2. **[Assignments]** タブを開き、利用対象となるグループをアプリにAssignします。

---

## ステップ4：Cloudflare Access ポリシーの作成

Oktaとの連携が完了したので、実際にアクセス制御ポリシーを作成します。

1. Cloudflare Zero Trustの **[Access] > [Applications]** を開き、**[Add an application]** をクリック。
2. Self-hosted（または該当する種別）を選択し、保護したいアプリケーションのURLを入力。
3. **Policies** の設定で、ルールを作成します。
   - Action: `Allow`
   - Include セクション:
     - Selector: `SAML Groups` （または `Okta Groups`）
     - Value: OktaのPush Groupsで送信したグループ名（サジェストされます）

これで、対象URLへアクセスしたユーザーはOkta認証を求められ、指定グループに属している場合のみアクセス可能になります。

---

## 💀 実践でハマった罠とトラブルシューティング

公式ドキュメント通りに進めても、見落としがちな仕様やエラーが多数存在します。構築時に実際に遭遇したトラブルとその解決策をまとめました。

### 罠1：SCIM接続テストで `Invalid Unique Field Name` エラー
OktaのSCIM連携（Integration）設定でテストボタンを押した際、以下のエラーが出ることがあります。
`Invalid Unique Field Name for the SCIM Connector. Valid Standard Field Names : [...]`
- **原因**: ユーザーの識別子（Unique identifier field for users）に `email` などを指定している。
- **解決策**: SCIMのスキーマ仕様上、ここは **`userName`** （途中のNは大文字）と入力するのが正解です。

### 罠2：SCIM接続テストで `Empty Authentication UserName` エラー
- **原因**: Authentication Modeがデフォルトの `Basic Auth`（ID/PW方式）のままになっている。CloudflareのSCIMはトークン認証を要求します。
- **解決策**: Authentication Modeのドロップダウンを **`HTTP Header`** に変更し、Authorization欄にSCIM Secretを入力してください。

### 罠3：SCIMでPushしたのに、Cloudflareの管理画面にユーザーが表示されない！
Okta側でエラーなくユーザーをAssign（Push）したにも関わらず、Cloudflare Zero Trustの `My Team > Users` 一覧に対象ユーザーがいつまで経っても現れないという現象が起きます。
OktaのTaskログを見ると `Matching user not found` のエラーが出ていることも。
- **原因**: **Cloudflareの仕様です。** Cloudflare Accessでは、IdPからプロビジョニングされても、**そのユーザーが一度もCloudflare Access（保護されたアプリ）にSAMLログインしていない場合、ダッシュボード上にユーザーとして表示されません。**
- **解決策**: エラーは無視して大丈夫です。ユーザー自身にSAML経由で初回ログインを行わせてください。ログインが成功した瞬間に、Users一覧に初めて実体が表示されます。なお、ユーザー実体が見えなくても、**Push Groupsで送ったグループ情報は即座にポリシーのサジェストに表示され利用可能**です。

### 罠4：Cloudflareの管理画面（`dash.cloudflare.com`）のSSOがOktaでエラーになる
「Zero TrustのIdP連携はできたから、IT管理者がCloudflareの管理コンソールにログインする時もOktaのSSOボタンを使おう！」と思って `Log in with SSO` を試すとエラーになります。
- **原因**: Cloudflare管理コンソール（Dashboard）自体のSSO機能は、**Enterpriseプラン限定の別機能**です。Zero Trust（Access）のSSOは無料枠でも使えますが、ダッシュボード管理者のSSOは別物という罠です。
- **解決策**: Cloudflareのダッシュボード管理者は、手動でメールアドレスを招待し、ID/PW（＋Cloudflare側のMFA）で個別にログインさせる必要があります。

### 罠5：Cloudflare管理者への招待メールが「有効期限切れ」で登録できない
前述の通り管理者は手動で招待しますが、届いた招待メールのリンクを踏むと「有効期限切れ(Expired)」になったり、シークレットモードで開くと登録失敗後に「アカウントが削除されました」という謎のメールが届くことがあります。
- **原因**: 
  1. 企業メールのセキュリティ（Defender等）が自動でリンクを事前スキャンし、ワンタイムトークンを消費してしまっている。
  2. シークレットモードで開いたため、Cloudflareの強力なBot対策（Turnstile）に引っかかり登録プロセスが強制遮断された。
- **解決策**: 招待リンクを踏む前に、**招待されるユーザーが自身で `dash.cloudflare.com` から通常通り無料アカウントを作成し、ログインした状態にしておきます**。その後、管理者が招待メールを再送し、ユーザーはログイン済みの通常ブラウザ（シークレットモード不可）で招待リンクをクリックします。これで確実に紐付けが完了します。

---

## おわりに

OktaとCloudflare AccessのSAML/SCIM連携は、設定項目が多く、特有の仕様（特に初回ログインまでユーザーが見えない点）に翻弄されがちですが、一度構築してしまえば非常に強力でメンテナンスフリーなゼロトラスト環境が手に入ります。

特にSCIMを使ったグループのプロビジョニングを適切に行うことで、Okta上の人事異動や退職処理が、即座に社内システム群のアクセス権剥奪へと連動する、堅牢なセキュリティ統制が実現可能です。ぜひお試しください。