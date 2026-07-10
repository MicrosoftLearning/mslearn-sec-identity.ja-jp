# ラボ 3A セットアップ - SharePoint サイトの構成

このフォルダーには、"**SC-500 ラボ 3A: Microsoft Purview を使用して AI データのリスクを明らかにする**" のセットアップ資料が含まれます。

## 概要

ラボ 3A には、**sc500-ai-datastore** という名前の事前構成済み SharePoint サイトが必要です。
- 過剰共有の構成 ("外部ユーザーを除くすべてのユーザー" と共有)
- 人事と財務のデータを含むサンプル ドキュメント
- ドキュメントへの秘密度ラベルの適用はなし
- Copilot 対話シグナル (通常の Copilot の使用によって生成)

## セットアップ オプション

### オプション 1: PowerShell スクリプト (推奨)

提供されている PowerShell スクリプトを使って、自動セットアップを行います。

#### 前提条件
- PowerShell 7.x 以降
- SharePoint 管理者またはグローバル管理者の資格情報
- インターネット接続

#### 手順

1. 管理者として PowerShell 7 を開きます

2. 次のディレクトリに移動します:
   ```powershell
   cd "C:\path\to\Allfiles\Lab-3A"
   ```

3. セットアップ スクリプトを実行します。
   ```powershell
   .\lab-3a-setup.ps1 -TenantUrl "https://YOUR-TENANT.sharepoint.com"
   ```
   `YOUR-TENANT` は、ご自分のテナント名に置き換えます。

4. 求められたら、グローバル管理者の資格情報を使って認証を行います

5. スクリプトが完了するまで待ちます (約 2 - 3 分)

#### スクリプトで行われること
- ✓ `/sites/sc500-ai-datastore` で SharePoint コミュニケーション サイトを作成する
- ✓ "外部ユーザーを除くすべてのユーザー" と共有するようにサイトのアクセス許可を構成する
- ✓ 3 つのサンプル ドキュメントをアップロードする:
  - `Employee_Salary_Report_2026.txt` (SSN、給与、PII を含む人事データ)
  - `Q2_Financial_Statement_2026.txt` (銀行口座、クレジット カードを含む財務データ)
  - `HR_Benefits_Guide_2026.txt` (報酬データ、個人情報)
- ✓ 秘密度ラベルが適用されないようにする

### オプション 2: 手動セットアップ

PowerShell スクリプトを使いたくない場合は、以下の手順に従って手作業で行います。

#### 1. SharePoint サイトを作成する

1. [SharePoint 管理センター](https://admin.microsoft.com/sharepoint)にサインインします
2. **[サイト]** > **[アクティブなサイト]** を選びます
3. **[作成]** > **[コミュニケーション サイト]** をクリックします
4. 構成する:
   - **サイト名**: SC500 AI Data Store
   - **サイト アドレス**: `/sites/sc500-ai-datastore`
   - **説明**: Pre-seeded SharePoint site for SC-500 Lab 3A
5. **[完了]** をクリックします。

#### 2. 過剰共有のアクセス許可を構成する

1. 新しいサイト `https://YOUR-TENANT.sharepoint.com/sites/sc500-ai-datastore` に移動します
2. **[設定]** (歯車アイコン) > **[サイトのアクセス許可]** をクリックします
3. **[サイトの共有]** をクリックします
4. 共有ダイアログで次のようにします。
   - 「**外部ユーザーを除くすべてのユーザー**」と入力します
   - アクセス許可レベルを **[読み取り]** に設定します
   - [メールの送信] をオフにします
5. **[共有]** をクリックします

#### 3. サンプル ドキュメントをアップロードする

1. サイトの **Documents** ライブラリに移動します
2. 次の内容で 3 つのテキスト ドキュメントを作成します。

**Employee_Salary_Report_2026.txt**:
```
CONFIDENTIAL - HUMAN RESOURCES

Employee Salary Report - Q2 2026

Employee ID: 10234
Name: Sarah Johnson
SSN: 123-45-6789
Department: Engineering
Position: Senior Software Engineer
Salary: $145,000
Bonus: $15,000

Employee ID: 10567
Name: Michael Chen
SSN: 987-65-4321
Department: Finance
Position: Financial Analyst
Salary: $95,000

This document contains personally identifiable information and financial data.
```

**Q2_Financial_Statement_2026.txt**:
```
CONFIDENTIAL - FINANCIAL REPORT

Quarterly Financial Statement - Q2 2026

Revenue: $4.5M
Operating Expenses: $2.1M
Net Income: $2.4M

Bank Account: 1234-5678-9012-3456
Routing Number: 123456789
Credit Line: $1,000,000

This document contains sensitive financial information.
```

**HR_Benefits_Guide_2026.txt**:
```
INTERNAL USE ONLY

Employee Benefits Guide 2026

Salary Ranges by Level:
  Level 1 (Entry): $55,000 - $70,000
  Level 2 (Mid): $75,000 - $95,000
  Level 3 (Senior): $100,000 - $140,000

This document contains compensation data and personal information.
```

3. 3 つのファイルすべてを Documents ライブラリにアップロードします
4. **重要**: これらのドキュメントには秘密度ラベルを適用しないでください

#### 4. Copilot 対話シグナルを生成する (省略可能)

現実的な Copilot 対話データを作成するには:

1. サイトの作成後、インデックスが作成されるまで 1 から 2 時間待ちます
2. Microsoft 365 Copilot を使ってサイト コンテンツのクエリを実行します。
   - "sc500-ai-datastore サイトで従業員の給与レポートを要約します"
   - "sc500-ai-datastore にはどのような財務情報が格納されていますか?"
   - "ai-datastore サイトの人事ドキュメントを示してください"
3. これらのクエリから、Purview DSPM に表示される対話シグナルが生成されます

## セットアップ後のタイムライン

セットアップが完了した後:

| 期間 | 起こること |
|-----------|--------------|
| 即時 | すべてのユーザーがサイトにアクセスできます |
| 1-2 時間 | SharePoint のインデックス作成が完了します |
| 4 から 6 時間 | Purview が初期スキャンを始めます |
| 24 - 48 時間 | DSPM の完全なシグナルがダッシュボードに表示されます |

**注意**: ラボ環境の場合は、ラボの演習をすぐに続けることができます。 一部の DSPM シグナルは完全に設定されるまで時間がかかる場合がありますが、サイトの構造とアクセス許可はすぐに評価できるようになります。

## トラブルシューティング

### PowerShell モジュールの問題

スクリプトによる `PnP.PowerShell` のインストールが失敗する場合:

```powershell
# Manually install the module
Install-Module -Name PnP.PowerShell -Scope CurrentUser -Force -AllowClobber

# If that fails, use this alternative:
Install-Module -Name PnP.PowerShell -Scope CurrentUser -Force -SkipPublisherCheck
```

### アクセス許可の問題

"アクセス拒否" エラーを受け取る場合:
- 自分が **SharePoint 管理者**または**グローバル管理者**であることを確認します
- 正しいテナント URL に接続していることを確認します
- 管理者として PowerShell を実行してみます

### サイトが既に存在する

前のラボ実行のサイトが既に存在する場合:
- スクリプトによって、既存のサイトの更新を求められます
- または、最初に SharePoint 管理センターからサイトを手動で削除します
- または、スクリプトのパラメーターで別のサイト URL を使います

## ARM テンプレートがない理由

Azure Resource Manager (ARM) JSON テンプレートを使うラボ 2A、2B、2C とは異なり、ラボ 3A では ARM テンプレートを使用できません。それは次のような理由のためです。

- **SharePoint サイトは Microsoft 365 のリソースであり**、Azure のリソースではありません
- ARM テンプレートでは、Azure のリソース (VM、ストレージ、ネットワークなど) のみがデプロイされます
- SharePoint の構成には、PowerShell (PnP.PowerShell) または Microsoft Graph API が必要です
- このため、ラボ 3A では `.json` テンプレートではなく `.ps1` スクリプトを使います

## クリーンアップ

ラボの完了後にサイトを削除するには:

### PowerShell による方法
```powershell
Connect-PnPOnline -Url "https://YOUR-TENANT-admin.sharepoint.com" -Interactive
Remove-PnPTenantSite -Url "https://YOUR-TENANT.sharepoint.com/sites/sc500-ai-datastore" -Force
```

### 手動による方法
1. [SharePoint 管理センター](https://admin.microsoft.com/sharepoint)に移動します
2. **[サイト]** > **[アクティブなサイト]** を選びます
3. **sc500-ai-datastore** を見つけます
4. チェックボックスをオンにしてから、**[削除]** をクリックします
5. 削除の確定

## サポート

問題または質問がある場合:
- `Instructions\Labs\Lab-3A-Purview-AI-Risks.md` でラボの手順を確認します
- PowerShell の実行ポリシーを調べます: `Get-ExecutionPolicy` (RemoteSigned または Unrestricted である必要があります)
- テナント URL の形式を確認します: `https://TENANT.sharepoint.com` (末尾のスラッシュなし)

---

**ラボ**: SC-500 ラボ 3A - Microsoft Purview を使用して AI データのリスクを明らかにする  
**セットアップの種類**: PowerShell (SharePoint Online)  
**推定セットアップ時間**: 3 - 5 分  
**DSPM シグナルを利用可能**: 24 - 48 時間
