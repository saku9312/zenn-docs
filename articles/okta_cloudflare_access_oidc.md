---
title: "沼からの脱出！Okta × Cloudflare AccessをOIDCで連携し、グループ制御するまでの全記録"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["okta", "cloudflare", "zerotrust", "oidc", "セキュリティ"]
published: true
---

# はじめに

社内アプリケーションへのアクセスをセキュアにするため、**Cloudflare Zero Trust (Access)** と **Okta** を連携させたZTNA（ゼロトラストネットワークアクセス）環境の構築に取り組みました。

当初はSAML 2.0とSCIMプロビジョニングを用いた連携を試みていましたが、Oktaのグループ情報をCloudflare側のポリシーで判定させる部分で数々の仕様の壁にぶつかり、最終的に**OIDC (OpenID Connect)** を用いた連携に切り替えることで、シンプルかつ確実なグループアクセス制御を実現できました。

しかし、その道のりは平坦なものではありませんでした。「404エラー」「リダイレクトループ（ERR_TOO_MANY_REDIRECTS）」「SCIMの謎の500エラー」、そして「最後の最後でログインできない罠」など、ありとあらゆる地雷を踏み抜きました。

この記事では、OktaとCloudflare AccessをOIDCで連携する**正しい手順**と、私がハマった**具体的なエラー（躓きポイント）とその解決策**を、備忘録も兼ねて超詳細に解説します。

これから同様の環境を構築する方の、トラブルシューティングのバイブルになれば幸いです。

---

# なぜSAMLではなくOIDCを選んだのか？

本題に入る前に、なぜSAML連携を途中で諦め、OIDC連携に切り替えたのかを説明しておきます。
最大の理由は**「グループ情報（クレーム）の受け渡しと、アクセスポリシー判定の仕様の複雑さ」**です。

* **SAML連携の辛いところ**:
  Okta公式のCloudflare OINアプリを使うと、グループ情報はSAMLアサーション（XML）内には含まれず、別途SCIM（Push Groups）を使ってCloudflare側に事前に同期させる必要があります。しかし、設定を少しでも間違えると「SAML Groups」や「Okta Groups」のポリシーセレクターが正しく動作せず、`That account does not have access.` と弾かれ続ける泥沼にハマります。
* **OIDC連携のメリット**:
  OIDCであれば、ログイン時にOktaから発行される **IDトークンの中に直接 `groups` クレーム（所属グループ一覧）を埋め込んで渡す** ことができます。SCIMのような事前同期の仕組みに依存せず、トークンの中身だけでアクセス制御（認可）が完結するため、アーキテクチャが非常にシンプルになります。

---

# 構築手順：OktaとCloudflare AccessのOIDC連携

まずは、罠を回避した「最短距離の正しい構築手順」を解説します。

## 1. Okta側：OIDCアプリケーションの作成

Oktaの管理コンソールから、Cloudflareと連携するためのOIDCアプリを新規作成します。

1. **[Applications] > [Applications]** を開き、`Create App Integration` をクリック。
2. **Sign-in method**: `OIDC - OpenID Connect` を選択。
3. **Application type**: `Web Application` を選択して `Next` へ。
4. **New Web App Integration** の設定画面で以下を入力します。
   * **App integration name**: `Cloudflare Access (OIDC)` など任意の名前
   * **Grant type**: `Authorization Code` にチェック
   * **Sign-in redirect URIs**: 
     👉 **`https://<あなたのチームドメイン>.cloudflareaccess.com/cdn-cgi/access/callback`**
     *(※ここが第一の罠ポイントです。後述します)*
   * **Assignments**: `Skip group assignment for now`（後で割り当てます）
5. `Save` を押して作成を完了します。

作成後、**[General]** タブに表示される **Client ID** と **Client secret** をメモしておきます。

## 2. Okta側：IDトークンにグループ情報（Groups Claim）を含める設定

OIDCのIDトークン内に、ユーザーが所属するOktaのグループ一覧を含める設定を行います。これがないと、Cloudflare側でグループベースのアクセス制御ができません。

1. 作成したアプリの **[Sign On]** タブを開きます。
2. 画面を下へスクロールし、**[OpenID Connect ID Token]** セクションの `Edit` をクリック。
3. **Groups claim filter** を以下のように設定します。
   * ドロップダウン: **`groups`** と入力
   * フィルター条件: **`Matches regex`**
   * 値: **`.*`** （※正規表現で「すべて」を意味するピリオド＋アスタリスク）
4. `Save` をクリックします。

## 3. Cloudflare側：IdP (Identity Provider) の登録

次に、Cloudflare Zero Trust側の管理画面でOktaをOIDCプロバイダーとして登録します。

1. **Cloudflare Zero Trust** (`one.dash.cloudflare.com`) にログイン。
2. **[Settings] > [Authentication]** (または Integrations > Identity providers) を開き、`Add new` をクリック。
3. **OIDC** を選択します。
4. 以下の通り設定値を入力します。
   * **Name**: `Okta OIDC` など
   * **App ID**: メモした `Client ID`
   * **Client secret**: メモした `Client secret`
   * **Okta account URL**: `https://<あなたのOktaドメイン>.okta.com`
     👉 *(※ここが第二の罠ポイントです。絶対に `-admin` を含めないでください)*
   * **OIDC Claims (オプション)**: **`groups`** と入力
     *(※これでOktaのトークンからgroups属性を読み取ります)*
5. `Save` して保存します。
6. 一覧に戻ったら、追加したプロバイダーの横にある `Test` ボタンを押し、Oktaのログイン画面が表示されて認証が成功すれば連携完了です。

## 4. 忘れてはいけない最重要手順：Oktaアプリのユーザー割り当て

**【重要】** アプリを作り直した際に最も忘れやすいステップです。
Okta側で、このアプリケーションを利用できるユーザー（またはグループ）を割り当てます。

1. Okta管理画面で、作成したOIDCアプリの **[Assignments]** タブを開きます。
2. `Assign` > `Assign to Groups` (または People) をクリック。
3. ZTNA経由でアクセスさせたいユーザーが所属するグループを選択し、割り当てを完了します。

## 5. Cloudflare側：アクセスポリシーの作成

最後に、保護したい社内アプリケーション（Cloud Runや自社サーバーなど）へのアクセスルールを設定します。

1. Cloudflare Zero Trust の **[Access] > [Applications]** から、対象アプリの `Edit` を開きます。
2. **[Policies]** タブからポリシーを作成・編集します。
3. **Configure rules** の **Include** ルールを以下のように設定します。
   * **Selector**: **`OIDC Claims`**
     👉 *(※「Okta Groups」ではありません。第三の罠ポイントです)*
   * **Claim name**: **`groups`**
   * **Claim value**: **`<Okta上で作成した許可したいグループ名>`** （例: `ZTNA-Users`）
4. `Save policy` で保存します。

これで、正しい手順での構築は完了です。ブラウザのシークレットウィンドウでアプリのURLにアクセスし、Oktaのログインを経て画面が表示されれば大成功です。

---

# 💣 いっぱい躓いた！ハマった罠と具体的な解決策

ここからは、私がこの構築過程で実際に踏み抜いた**「数々のエラーとその原因・解決策」**を共有します。公式ドキュメント通りに進めたつもりでも、少しの設定のズレで致命的なエラーが発生します。

## 罠1: 終わらないリダイレクトループと「404」の迷宮（URL設定ミス）

**【事象】**
Cloudflare側のアプリURLにアクセスすると、Oktaへ飛んでログインは成功するものの、その後ブラウザが何度も画面遷移を繰り返し、最終的に `ERR_TOO_MANY_REDIRECTS`（リダイレクトが多すぎます）と表示されてクラッシュする。
また、URLの末尾を適当にいじると `404 ページが見つかりません` というCloudflareのエラー画面が出る。

**【原因】**
Okta側の `Sign-in redirect URIs` に、**誤ったエンドポイント（窓口）**を指定していました。

私が最初に指定してしまったのは以下のURLです。
❌ `https://<ドメイン>.cloudflareaccess.com/cdn-cgi/access/app`
❌ `https://<ドメイン>.cloudflareaccess.com/cdn-cgi/access/saml`

`/app` は人間が見るための「Cloudflareアプリポータル画面」であり、認証データを受け取る窓口ではありません。Oktaがここに認証成功データを投げ込んでも、Cloudflareは「ここは窓口じゃない。未ログインだからもう一度Oktaへ行け」と追い返してしまい、無限ループが発生していました。

**【解決策】**
SAMLでもOIDCでも、Cloudflare Accessが認証データを受け取るための**正しいコールバックURLは一つだけ**です。

⭕ **`https://<あなたのチームドメイン>.cloudflareaccess.com/cdn-cgi/access/callback`**

Okta側の `Sign-in redirect URIs` をこの完全なURL（末尾が `/callback` ）に修正することで、データのキャッチボールが正しく行われ、ループがピタッと止まりました。

## 罠2: Okta側の「404 リクエストしたページが見つかりませんでした」

**【事象】**
Cloudflare側でOkta (OIDC) のIdP設定を行い、`Test` ボタンを押した瞬間、Oktaの画面に飛んで以下のように表示される。

> **404 ページが見つかりません**
> リクエストしたページが見つかりませんでした。

**【原因】**
Cloudflare側に入力する **「Okta account URL（テナントURL）」に、`-admin` を含めてしまっていたこと**が原因でした。

Oktaの管理コンソールを操作しているとき、ブラウザのアドレスバーは `https://dev-xxxxxx-admin.okta.com` のようになっています。これをそのままコピーして貼り付けてしまうのが最大の罠です。

`-admin` がつくURLは「管理者専用サーバー」であり、OIDCの認証エンドポイント（`/oauth2/v1/authorize` など）が存在しません。そのため、Cloudflareがそこへユーザーを飛ばそうとしても、Okta側が「そんなページはない」と404を返してしまいます。

**【解決策】**
CloudflareのIdP設定の `Okta account URL` から、**`-admin` と末尾のスラッシュを削除**します。

❌ `https://dev-xxxxxxx-admin.okta.com`
❌ `https://dev-xxxxxxx.okta.com/admin/`
⭕ **`https://dev-xxxxxxx.okta.com`**

これを直すだけで、正常なOktaのログインフォームが表示されるようになります。

## 罠3: ポリシー設定で「Okta Groups」を選ぶと謎のエラーが出る

**【事象】**
Cloudflare Accessのポリシー設定画面で、グループを指定しようと Selector から `Okta Groups` を選ぶと、画面上に赤い通知で以下のようなエラーが出る。

> `Failed to retrieve SCIM groups due to an unexpected error`

**【原因】**
Cloudflareの仕様に対する私の勘違いでした。
OIDC連携を行っているのに、ポリシー側で `Okta Groups` を使おうとしたことが原因です。

`Okta Groups` というセレクターは、「SCIM（プロビジョニングAPI）を使って裏側でCloudflareに同期・保存された名簿データ」を読みに行こうとする機能です。今回はOIDC連携に切り替えたためSCIMの同期設定をしておらず、Cloudflareが「名簿データが存在しない！」とパニックを起こして500エラーを出していました。

途中で「Okta Provisioning Agents（オンプレ用エージェント）」をインストールしろという画面に迷い込んだのも、このSCIMの幻影を追ってしまったためです。

**【解決策】**
OIDC連携においてSCIMは**不要**です。
ログイン時に飛んでくるトークンの中身（クレーム）を直接判定すればよいため、ポリシーの Selector は **`OIDC Claims`** を使用します。

* **Selector**: `OIDC Claims`
* **Claim name**: `groups`
* **Claim value**: `<Oktaのグループ名>`

これでSCIMエラーを完全に回避しつつ、グループ制御が可能になります。

## 罠4: Oktaでパスワードは合っているのに「サインインできません (VERIFICATION_ERROR)」

**【事象】**
Cloudflare経由でOktaのログイン画面が表示され、正しいIDとパスワードを入れたのに画面に「サインインできません」と表示される。Google Workspace (GWS) へのSSOログインは同じパスワードで成功しているのに、Cloudflareのときだけ弾かれる。

Oktaの `System Log` を確認すると、以下のようなログが残っていた。
> `FAILURE: VERIFICATION_ERROR`

**【原因】**
アプリケーションごとに設定されている **Oktaの認証ポリシー（Authentication Policy）の違い** が原因でした。

GWSアプリ側は「パスワードのみ」でログインできるポリシーになっていたのに対し、新しく作成したCloudflare (OIDC) アプリ側にはデフォルトの厳しいポリシーが適用されており、**「MFA（多要素認証 / Any 2 factors）」が必須**になっていました。
テスト用アカウントにMFAを設定しておらず、パスワードしか入力しなかったため、Okta側が「要求しているセキュリティレベルに達していない」として認証を拒否していました。

**【解決策】**
Oktaの管理画面から、Cloudflareアプリに紐づく認証ポリシーを一時的に緩和しました。

1. **[Security] > [Authentication Policies]** を開く。
2. 該当アプリのルールの `User must authenticate with` を **`Password only`**（または `Any 1 factor type`）に変更して保存。

本番運用ではMFAを必須にすべきですが、構築時の切り分けテストにおいては、このポリシーの壁に気づけるかどうかが重要でした。

## 罠5: 【最大の盲点】すべての設定が完璧なのにアクセスできない

**【事象】**
404も直し、ループも直し、ポリシーも `OIDC Claims` にした。Oktaでのパスワードログインも成功し、ついにCloudflare Accessまで処理が戻ってきた！……と思ったら、Cloudflareの画面で無慈悲なエラーが出た。

> **That account does not have access.**
> (そのアカウントにはアクセス権がありません)

**【原因】**
これぞまさに灯台下暗し。
SAMLでドツボにハマってOIDCアプリを**新しく作り直した際、Okta側で「そのアプリケーションをユーザー（グループ）に割り当てる (Assignments)」作業を忘れていた**のです。

Oktaは、ユーザーがそのアプリに割り当てられていない場合、OIDCのログインフローの途中でエラーを返したり、必要な情報をCloudflareに渡さなかったりします。結果としてCloudflare側は「認証は通ったように見えるが、認可に必要な情報が足りない」と判断し、アクセスをブロックしていました。

**【解決策】**
「手順4」で記載した通り、Okta管理画面の **[Applications] > 対象アプリ > [Assignments]** タブを開き、対象のグループ（またはユーザー）にアプリをアサインしました。

これを設定した直後、もう一度アクセスを試みると……見事、保護された社内アプリケーションの画面が表示されました！！

---

# まとめ：ゼロトラスト構築で学んだこと

OktaとCloudflare Accessの連携は、うまくいけば非常に強力でシームレスなZTNA環境を提供してくれます。しかし、以下のポイントを少しでも見落とすと、簡単に泥沼にハマります。

1. **OIDC連携ならグループ制御は `OIDC Claims` を使う！** （SCIMや `Okta Groups` にこだわらない）
2. **コールバックURLは `/callback` で終わる！** （`/app` はポータル用）
3. **OktaのテナントURLに `-admin` を絶対に入れない！**
4. **「認証（Authentication）」と「認可（Authorization）」のエラーを切り分ける！**（OktaのSystem Logと、CloudflareのAccess Logの両方を見る）
5. **アプリを作ったら、最後に必ずユーザーへ割り当てる（Assign）！**

同じようにSAML連携やグループ制御で沼にハマっている方は、ぜひ一度OIDC連携へ切り替えて、この手順と罠のリストを確認してみてください。

この記事が、どこかの誰かの「404エラー」や「リダイレクトループ」を止める助けになれば嬉しいです！