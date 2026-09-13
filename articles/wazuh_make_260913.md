---
title: "ProxmoxへのWazuh構築手順（OVA展開→VMDKインポート）"
emoji: "🐺"
type: "tech"
topics: ["wazuh", "proxmox", "windows11", "zenn", "security"]
published: true
---

## はじめに

セキュリティ監視・SIEMツールとして代表的な **Wazuh** を、Proxmox VE 上に OVA（仮想マシンイメージ）を使って最速で構築し、Windows 11 クライアントのセキュリティ状態やパスワード状況を監視する環境を構築しました。

本記事では、OVAのダウンロードから7-Zipでの展開、Proxmoxへの `vmdk` インポート（`qm importdisk`）、仮想マシン設定、Wazuh初期セットアップ、そして Windows 11 Agent の導入・動作検証までの全手順を詳細に解説します。

---

## 前提環境

| 項目 | 詳細・バージョン |
| :--- | :--- |
| **ハイパーバイザ** | Proxmox VE 8.x |
| **Wazuh サーバー** | Wazuh Virtual Appliance (OVA版) |
| **作業端末** | Windows 11 (7-Zip, SCPクライアント導入済み) |
| **監視対象クライアント** | Windows 11 Enterprise / Pro |
| **ネットワーク** | 全マシン同一 L2 セグメント (ローカルLAN) |

---

## 全体作業フロー

1. **Wazuh OVA の取得と解凍 (Windows)**
2. **`vmdk` ファイルの Proxmox サーバーへの転送**
3. **Proxmox 上での仮想マシン (VM) 作成と既存ディスク削除**
4. **`qm importdisk` による `vmdk` の取り込みと初期設定**
5. **VM の起動・Wazuh 初期設定・Web UI サインイン確認**
6. **Windows 11 への Wazuh Agent 導入**
7. **Wazuh Dashboard でのパスワード状況・セキュリティ状態の確認**

---

## Step 1: Wazuh OVA の取得と Windows での解凍

Wazuh 公式サイトから Virtual Appliance (OVA) をダウンロードし、Windows 端末上で `vmdk` ファイルを取り出します。

> ⚠️ **【重要】ProxmoxでのOVA直接インポート時のエラー400について**  
> Proxmox VE の Web UI や CLI から `.ova` ファイルを直接インポートしようとすると、OVF定義の互換性問題により **「HTTP Error 400 (Parameter verification failed)」** 等のエラーが発生して失敗します。  
> そのため、本手順のように **Windows側で 7-Zip を使って `.ova` を解凍し、内部の `.vmdk` ファイルを直接取り出してインポートする手法** が最も確実な回避策となります。

### 1-1. OVA ファイルの入手
公式ドキュメントまたはダウンロードページから、最新の Wazuh OVA ファイルをダウンロードします。

![](https://static.zenn.studio/user-upload/38bc46edb7d7-20260913.png)

### 1-2. 7-Zip による展開
OVA ファイルはアーカイブ形式（tar）のアーカイブです。Windows 上で **7-Zip** を使用して解凍します。

1. ダウンロードした `.ova` ファイルを右クリックします。
2. `7-Zip` > `展開...` を選択し、適当なフォルダへ展開します。

![](https://static.zenn.studio/user-upload/1954f70ac48f-20260913.png)

3. 展開されたファイル群の中から、システムディスクとなる **`.vmdk` ファイル**（例: `wazuh-vmdk-disk1.vmdk`）を確認します。

![](https://static.zenn.studio/user-upload/10827f212af4-20260913.png)

---

## Step 2: Proxmox サーバーへの vmdk 転送

抽出した `vmdk` ファイルを、WinSCP や PowerShell の `scp` コマンドを使用して Proxmox VE サーバーへ転送します。

### 2-1. 転送先ディレクトリの準備 (Proxmox CLI)
Proxmox サーバーに SSH 接続し、一時作業用のディレクトリを作成します。

```bash
mkdir -p /vmdk
```

### 2-2. SCP コマンドによるファイル転送 (Windows PowerShell)
Windows の PowerShell から以下のコマンドを実行し、ファイルを転送します。

```powershell
scp C:\path\to\extracted\wazuh-disk1.vmdk root@<Proxmox-IP>:/vmdk/wazuh.vmdk
```

![](https://static.zenn.studio/user-upload/144e28806ebe-20260913.png)

![](https://static.zenn.studio/user-upload/ca8dbf654dc7-20260913.png)

---

## Step 3: Proxmox 上での VM 作成と vmdk インポート

Proxmox の CLI ツール `qm` を利用して、ハードディスクを入れ替える形で VM を作成します。

### 3-1. ダミー VM の作成 (Proxmox Web UI)

1. Proxmox Web UI で **「Create VM」** をクリックします。
2. **General**: VM ID（例: `303`）と Name（例: `wazuh-server`）を入力します。

![](https://static.zenn.studio/user-upload/c2e58057a432-20260913.png)

3. **OS**: 「メディアを使用しない」を選択します。

![](https://static.zenn.studio/user-upload/634ef8b0e112-20260913.png)

4. **System**: デフォルトのまま進めます（必要に応じて Qemu Agent を有効化）。

![](https://static.zenn.studio/user-upload/415bb022dc4f-20260913.png)

5. **Disks**: 一度ダミーのハードディスクを作成します（後ほど削除します）。

![](https://static.zenn.studio/user-upload/bffc440b997a-20260913.png)

6. **CPU**: `4` ソケット以上を割り当てます。

![](https://static.zenn.studio/user-upload/d324d9d18586-20260913.png)

7. **Memory**: `8192` MB (8 GB) 以上を割り当てます。

![](https://static.zenn.studio/user-upload/f49db7af924f-20260913.png)

8. **Network**: 適切なブリッジ（`vmbr0` 等）を選択します。

![](https://static.zenn.studio/user-upload/944befed8383-20260913.png)


### 3-2. 既存ダミーハードディスクの削除

1. 作成した VM (ID: 303) の **Hardware** タブを開きます。

![](https://static.zenn.studio/user-upload/bf14bedfdc10-20260913.png)

2. 作成された **Hard Disk (scsi0)** を選択し、画面上部の **「Detach」** をクリックします。

![](https://static.zenn.studio/user-upload/d2898d0c7861-20260913.png)
![](https://static.zenn.studio/user-upload/f3e8133025bc-20260913.png)

3. 「Unused Disk 0」となった項目を選択し、**「Remove」** で削除します。

![](https://static.zenn.studio/user-upload/e930fdd6b070-20260913.png)
![](https://static.zenn.studio/user-upload/a2101750da21-20260913.png)


### 3-3. `qm importdisk` による vmdk のインポート (Proxmox CLI)
SSH ターミナルに戻り、`qm importdisk` コマンドを使って `vmdk` を VM にインポートします。

```bash
# 構文: qm importdisk <VMID> <vmdkパス> <ストレージ名>
qm importdisk 100 /vmdk/wazuh.vmdk local-lvm
```

![](https://static.zenn.studio/user-upload/91d85cb08ecc-20260913.png)

インポートが成功すると、`Successfully imported disk 'local-lvm:vm-303-disk-0'` という出力が表示されます。

![](https://static.zenn.studio/user-upload/28419256a704-20260913.png)
---

## Step 4: VM 設定の変更と起動

インポートしたディスクを仮想マシンに接続し、起動順序を変更します。

### 4-1. インポートしたディスクのアタッチ
1. Proxmox Web UI の **VM 100 > Hardware** を開きます。
2. **Unused Disk 0** をダブルクリック（または選択して Edit）します。
3. Bus/Device タイプ（`SCSI` や `VirtIO Block` など）を確認し、**「Add」** をクリックして接続します。

![](https://static.zenn.studio/user-upload/97be2ef94174-20260913.png)

### 4-2. ブート順序 (Boot Order) の変更
1. **VM 100 > Options** > **Boot Order** をダブルクリックします。
2. 追加したディスク（`scsi0` 等）の **Enabled** にチェックを入れ、リストの一番上にドラッグして移動します。
3. **「OK」** をクリックして保存します。

![](https://static.zenn.studio/user-upload/f9e067e3ea2b-20260913.png)

### 4-3. VM の起動
Proxmox Web UI から VM 100 の **「Start」** をクリックし、**Console** を開きます。

![](https://static.zenn.studio/user-upload/e0a16bda49ba-20260913.png)

---

## Step 5: Wazuh 初期セットアップと Web UI へのログイン

### 5-1. Wazuh の初期起動処理の確認
コンソール画面で Wazuh の起動シーケンスが完了するまで数分待ちます。正常に起動すると、ログインプロンプトとアクセス用の IP アドレスが表示されます。

### 5-2. 初期パスワードの確認
Wazuh OVA の初期パスワードはコンソール画面に表示されているか、あるいは内部のパスワードファイルに保存されています。必要に応じてコンソールからログインし、確認します。

```bash
# Wazuh VM内でのパスワード確認例 (必要に応じて)
cat /usr/share/wazuh-indexer/config/passwords.txt
# デフォルトはadmin/admin
```	

### 5-3. Wazuh Dashboard (Web UI) へのサインイン
1. ブラウザを開き、`https://<Wazuh-VMのIPアドレス>` にアクセスします。
2. 自己署名証明書のアラートが表示された場合は「詳細設定」からアクセスを続行します。
3. ログイン画面で ユーザー名 `admin` と確認したパスワードを入力してサインインします。

![](https://static.zenn.studio/user-upload/9c0643250725-20260913.png)

サインイン後、Wazuh Dashboard のトップ画面が表示されることを確認します。

![](https://static.zenn.studio/user-upload/359943c69cf7-20260913.png)

---

## Step 6: Windows 11 への Wazuh Agent 導入

監視対象となる Windows 11 端末に Wazuh Agent を導入し、マネージャー（Wazuh サーバー）と紐付けます。

### 6-1. エージェント追加コマンドの取得
1. Wazuh Dashboard 上部の **「Deploy new agent」** をクリックします。
2. **OS**: `Windows` を選択します。
3. **Server address**: Wazuh VM の IP アドレスを入力します。
4. **Agent name**: 任意の識別名（例: `win11-client`）を入力します。
5. 画面下に自動生成された PowerShell コマンドをコピーします。

![](https://static.zenn.studio/user-upload/8d73af0a08d1-20260913.png)
![](https://static.zenn.studio/user-upload/ff81d941c610-20260913.png)

### 6-2. Windows 11 での Agent インストール
Windows 11 端末側で **PowerShell を管理者権限** で起動し、コピーしたコマンドを実行します。

```powershell
# Agentのダウンロードとサイレントインストール
Invoke-WebRequest -Uri [https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi) -OutFile ${env:TEMP}\wazuh-agent.msi
msiexec.exe /i ${env:TEMP}\wazuh-agent.msi /q WAZUH_MANAGER='<Wazuh-VMのIP>'

# Wazuhサービスの起動
NET START Wazuh
```

![](https://static.zenn.studio/user-upload/c712bc49cd45-20260913.png)
![](https://static.zenn.studio/user-upload/fad0cc41f200-20260913.png)

### 6-3. エージェント接続確認
Wazuh Dashboard の **Endpoints > Agents** を確認し、追加した Windows 11 端末のステータスが **Active** になっていることを確認します。

---

## Step 7: Windows 11 のパスワード状況・セキュリティ状態の検証

Wazuh Agent が正常に認識された後、Wazuh の標準機能（SCA: Security Configuration Assessment および Security Events）を利用して、Windows 11 のパスワードポリシーやログイン状態を検証します。

### 7-1. パスワード設定・アカウントポリシーの診断 (SCA)
1. Wazuh Dashboard の左メニューから **Security Configuration Assessment (SCA)** を開きます。
2. 登録した Windows 11 エージェントを選択し、**「CIS Benchmark for Windows 11」** などのポリシーレポートを開きます。
3. 評価項目の中から **Password Policy** や **Account Lockout Policy** に関連するチェック結果を確認します。

* **最小パスワード長 (Minimum password length)**
* **パスワードの複雑性要件 (Password must meet complexity requirements)**
* **アカウントロックアウト閾値 (Account lockout threshold)**

> 💡 **確認できるポイント**: 不十分なパスワードポリシー（例：文字数制限が短い、失敗時のロックアウトが無効化されているなど）が「FAILED」として可視化されます。

### 7-2. ログイン試行・パスワード変更イベントの検知 (Security Events)
Windows 11 側でログイン失敗やパスワード変更を行った際、Wazuh 側にリアルタイムでイベントが届くか確認します。

1. Windows 11 側で意図的に誤ったパスワードを入力してログイン失敗を発生させます。
2. Wazuh Dashboard の **Security Events** タブを開き、フィルタに `agent.name: "win11-client"` を指定します。
3. Windows のセキュリティイベントログ（Event ID: `4625` ログイン失敗、Event ID: `4723` パスワード変更試行など）が正しくキャプチャされていることを確認します。

---

## 導入時の注意点・ハマりポイントまとめ

今回の構築で発生しやすいトラブルと対策をまとめました。

1. **ProxmoxでOVAを直接インポートしようとすると「エラー400」が発生する**
   * Web UIなどの直インポート機能や `qm importovf` を使うと、OVF定義の非互換により **HTTP Error 400 (Parameter verification failed)** が表示されます。本記事で解説した通り「7-Zipで解凍してvmdkを取り出し、`qm importdisk` を実行する」手順を踏むことで確実に回避できます。
2. **`qm importdisk` 実行時のストレージ指定エラー**
   * Proxmox のノードによってストレージ名（`local-lvm` や `local-zfs` など）が異なります。必ず `pvesm status` で有効なストレージ名を確認した上でコマンドを実行してください。
3. **起動時に「No bootable device found」と表示される**
   * VM の Options > **Boot Order** で、アタッチしたディスクが有効化されていない・または最優先になっていないことが原因です。
4. **Windows 11 Agent が Active にならない**
   * Windows 11 側の `firewalld` やネットワーク接続（プライベート/パブリック）により、Wazuh サーバー（ポート `1514/TCP`）への通信がブロックされている場合があります。`Test-NetConnection <Wazuh-IP> -Port 1514` で疎通を確認してください。

---

## おわりに

Proxmox において OVA ファイルを直接インポートする際のエラー 400 回避策として、`vmdk` ファイルを取り出して `qm importdisk` を実行する手法を用いることで、非常に安定して Wazuh サーバーを稼働させることができました。

また、Windows 11 に Wazuh Agent を導入したことで、単なるリソース監視にとどまらず、**パスワードポリシーのコンプライアンス状況や認証セキュリティイベントを可視化・統合管理** できる体制が整いました。次のステップとして、冒頭でご紹介した Grafana や Zabbix との連携を進めることで、インフラとセキュリティを跨いだ統合モニタリング環境が完成します。

Wazuhはここから、運用するために脆弱性情報の収集やアラート通知の設定、ログの長期保存など、さらに多くの機能を活用することが可能です。今後は、これらの機能を活用して、より高度なセキュリティ監視体制を構築していくこともやりたいと思います。