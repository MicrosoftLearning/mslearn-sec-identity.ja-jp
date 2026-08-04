# SC-500 ラボ 3C - セットアップ手順

## 概要

このフォルダーには、"**ラボ 3C: AI ゲートウェイと Foundry のセキュリティ コントロールを構成する**" のセットアップ リソースが含まれています。

## このフォルダー内のファイル

- **lab-3c-setup.json** - Azure portal で [カスタム テンプレートのデプロイ] を使ってラボ インフラストラクチャをデプロイするための ARM テンプレート
- **sc500-lab3c-apim-policy.xml** - API Management トークンのレート制限ポリシー (ラボの間に学生がこれを適用します)
- **generate-adversarial-traffic.ps1** - 素早い要求でレート制限をテストするための PowerShell スクリプト
- **README.md** - このファイル

## セットアップによってデプロイされるもの

`lab-3c-setup.json` ARM テンプレートでは、**sc500-lab3c-rg** リソース グループに以下のリソースがプロビジョニングされます。

### コア インフラストラクチャ
- **Azure OpenAI Service** (`sc500-lab3c-ai-{instanceId}`)
  - 10 TPM 容量の gpt-5.4-mini モデル デプロイ (GlobalStandard SKU)
  - 既定のコンテンツ フィルターで初期構成済み (カスタム ガードレールなし)
  
- **Azure AI Foundry ハブ** (`sc500-lab3c-hub-{instanceId}`)
  - Foundry プロジェクトのハブ ワークスペース
  
- **Azure AI Foundry プロジェクト** (`sc500-lab3c-foundry`)
  - 学生が Content Safety ガードレールを構成するプロジェクト ワークスペース
  
- **Azure API Management** (`sc500-lab3c-apim-{instanceId}`)
  - 従量課金レベルの APIM インスタンス
  - 事前登録済み API: `sc500-foundry-api`
  - **意図的な非セキュリティ保護状態:**
    - ❌ サブスクリプション キーの必要性なし (学生がこれを有効にします)
    - ❌ レート制限ポリシーなし (学生がこれを適用します)
    - ❌ Content Safety ガードレールなし (学生が Foundry でこれを作成します)

### サポート リソース
- **Log Analytics ワークスペース** (`sc500-lab3c-logs`)
- **Application Insights** (`sc500-lab3c-insights`)
- **ストレージ アカウント** (`sc500lab3c{instanceId}`) - AI ハブ/プロジェクト用
- **Key Vault** (`sc500-kv-{instanceId}`) - AI ハブ/プロジェクト用

## 展開指示

### オプション 1: Azure portal 経由でデプロイする (ホステッド ラボに推奨)

1. [Azure portal](https://portal.azure.com) にサインインする
2. **[カスタム テンプレートのデプロイ]** を検索します
3. **[エディターで独自のテンプレートを作成する]** を選択します
4. **[ファイルの読み込み]** をクリックして `lab-3c-setup.json` をアップロードします
5. **[保存]** をクリックします
6. パラメーターを構成します。
   - **サブスクリプション**: ラボのサブスクリプションを選択します
   - **場所**: Azure OpenAI がサポートされている Azure リージョンを選択します (例: 米国東部、西ヨーロッパ)
   - **ラボ インスタンス ID**: **空欄のままにして**、サブスクリプション ID から 8 文字のハッシュを自動生成します。 特定のオーバーライドが必要な場合のみ、値を入力します。
   - **発行元のメール アドレス**: 既定値のままにするか、カスタマイズします
   - **発行元名**: 既定値のままにするか、カスタマイズします
7. **[確認して作成]**、**[作成]** の順にクリックします

**デプロイ時間:** 約 10 - 15 分

### オプション 2: Azure CLI を使ってデプロイする

```bash
# Set variables
LOCATION="eastus"

# Deploy the template (labInstanceId defaults to a hash of the subscription ID)
az deployment sub create \
  --location $LOCATION \
  --template-file lab-3c-setup.json \
  --parameters location=$LOCATION
```

### オプション 3: PowerShell を使ってデプロイする

```powershell
# Set variables
$Location = "eastus"

# Deploy the template (labInstanceId defaults to a hash of the subscription ID)
New-AzSubscriptionDeployment `
  -Location $Location `
  -TemplateFile .\lab-3c-setup.json `
  -location $Location
```

## デプロイ後の構成

ARM テンプレートのデプロイが完了した後、**学生がラボを始める前に次のことを確認します。**

### 1. Azure OpenAI のデプロイを確認する
1. **[リソース グループ]** > **sc500-lab3c-rg** > **sc500-lab3c-ai-{instanceId}** に移動します
2. **[モデル デプロイ]** (または **[デプロイ]**) を選びます
3. **gpt-5.4-mini** がデプロイされていて、"成功" 状態を示していることを確認します

### 2. Azure AI Foundry プロジェクトの接続を構成する
ARM テンプレートによって AI Foundry プロジェクトが作成されますが、Azure OpenAI への接続の確認が必要な場合があります。

1. [Azure AI Foundry ポータル](https://ai.azure.com)に移動します
2. **sc500-lab3c-foundry** プロジェクトを選びます
3. 左側のナビゲーションで、**[設定]** または **[接続されたリソース]** を選びます
4. Azure OpenAI サービスへの接続が存在することを確認します
   - 見つからない場合は、**[+ 新しい接続]** → **[Azure OpenAI]** をクリックして、`sc500-lab3c-ai-{instanceId}` を選びます
5. **[モデル + エンドポイント]** に移動し、**gpt-5.4-mini** が表示されていることを確認します
6. **[安全性とセキュリティ]** > **[コンテンツ フィルター]** に移動し、gpt-5.4-mini にカスタム フィルターが割り当てられていないことを確認します ([既定] または [なし] が使われている必要があります)

### 3. API Management の API 構成を確認する
1. **sc500-lab3c-apim-{instanceId}** > **[API]** に移動します
2. **sc500-foundry-api** を選びます
3. **[設定]** タブに次のように表示されることを確認します。
   - ✅**サブスクリプションは必須です**:**必須ではありません** (これは意図的なもので、学生がそれを有効にします)
4. **[デザイン]** タブ > **[すべての操作]** > **[受信処理]** に次のように表示されていることを確認します。
   - ✅ `<base />` タグのみ (レート制限ポリシーなし - 学生がそれを適用します)

### 4. セキュリティ保護されていないエンドポイントをテストする (省略可能)
学生が始める前に、環境がセキュリティ保護されていない状態であることを確認するには、次のようにします。

1. APIM で、**[API]** > **sc500-foundry-api** > **[テスト]** に移動します
2. POST 操作を選びます
3. **サブスクリプション キー ヘッダーは何も追加しないでください** (匿名アクセスが機能することを確認します)
4. 要求本文に以下を貼り付けます。
   ```json
   {"messages": [{"role": "user", "content": "Hello"}], "max_tokens": 10}
   ```
5. ページの下部にある **[Send]**
6. 予想される結果: モデルが **HTTP 200** で応答します (匿名アクセスが許可されることを確認します)

## ラボのフロー

デプロイと検証が済むと、学生は以下のことを行います。

1. **セキュリティ保護されていない状態を確認する** - レート制限、認証、コンテンツ フィルターのいずれもないことを確認します
2. **トークン レート制限ポリシーを適用する** - `sc500-lab3c-apim-policy.xml` を使います
3. **サブスクリプション キーを要求する** - APIM の API 設定でサブスクリプション要件を有効にします
4. **構成されたゲートウェイをテストする** - 429 の応答と、キーがない場合の 401 を確認します
5. **Content Safety ガードレールを作成する** - Foundry でプロンプト シールドを構成します
7. **ガードレールをデプロイに適用する** - カスタム フィルターを gpt-5.4-mini に割り当てます
7. **AI 用 Defender を有効にする** - Microsoft Defender for Cloud で AI サービス用 Defender を有効にします

## トラブルシューティング

### 問題: Azure OpenAI のデプロイが AI Foundry に表示されない
**解決策:** AI Foundry プロジェクトの設定で接続を確認します。 `sc500-lab3c-ai-{instanceId}` を指す新しい Azure OpenAI 接続を追加します。

### 問題: API Management のデプロイが失敗する
**解決策:** 選んだ Azure リージョンで API Management の従量課金レベルがサポートされていることを確認します。 米国東部、西ヨーロッパ、または北ヨーロッパを試します。

### 問題: gpt-5.4-mini のデプロイが失敗する
**解決策:** サブスクリプションで Azure OpenAI のクォータを調べます。 このモデルには少なくとも 10 TPM の容量が必要です。 必要な場合は、クォータの増量を要求します。

### 問題: 学生が APIM API をテストするときに 403 を受け取る
**解決策:** APIM のバックエンド構成で `api-key` ヘッダーに Azure OpenAI の API キーが含まれることを確認します。 `sc500-lab3c-apim-{instanceId}` > **[バックエンド]** > **openai-backend** を調べます。

## クリーンアップ

ラボ ホストによって環境が自動的にリセットされる場合があります。 手動クリーンアップが必要な場合:

```bash
az group delete --name sc500-lab3c-rg --yes --no-wait
```

## 重要な注意点

- **意図的にセキュリティ保護されていない:** API は、サブスクリプション キー要件、レート制限、Content Safety ガードレールのいずれもなしでデプロイされます。 これは意図的なもので、学生はラボの間にこれらの制御を適用します。
- **モデルのバージョン:** テンプレートでは、`gpt-5.4-mini` のバージョン `2026-03-17` がデプロイされます (GA モデル、GlobalStandard デプロイ種類、2027 年 3 月に廃止)。
- **クォータ要件:** サブスクリプションに十分な Azure OpenAI クォータがあることを確認します。 gpt-5.4-mini の GlobalStandard デプロイでは、レベル 1 で 5,000 RPM と 5,000,000 TPM が提供されます。
- **Azure リージョンの利用可能性:** GlobalStandard のデプロイは、トラフィックをグローバルにルーティングします。 米国東部、西ヨーロッパ、または Azure OpenAI をサポートする他の Azure リージョンを使います。 [Azure OpenAI を利用可能な Azure リージョン](https://learn.microsoft.com/azure/ai-services/openai/concepts/models#model-summary-table-and-region-availability)に関する記事をご覧ください。

## ラボ リソース リファレンス

| リソースの種類 | リソース名 | パーパス |
|--------------|---------------|---------|
| リソース グループ | sc500-lab3c-rg | すべてのラボ リソースが含まれます |
| API Management | sc500-lab3c-apim-{instanceId} | レート制限と認証のための AI ゲートウェイ |
| Azure OpenAI | sc500-lab3c-ai-{instanceId} | gpt-5.4-mini モデルをホストする |
| AI Hub | sc500-lab3c-hub-{instanceId} | Foundry ハブ ワークスペース |
| AI プロジェクト | sc500-lab3c-foundry | Content Safety のための Foundry プロジェクト |
| モデル デプロイ | gpt-5.4-mini | 言語モデル エンドポイント |
| APIM API | sc500-foundry-api | Foundry モデルへの API ルーティング |

---

**最終更新日:** 2026 年 6 月 30 日  
**ラボ バージョン:** 1.0  
**対象者:** SC-500 の学生およびラボ環境管理者
