---
lab:
  title: Azure アプリ Services と API Management をセキュリティで保護する
  description: WAF の検出と防止の制御を使用し、App Services に対して Microsoft Entra 認証とネットワーク制限を構成し、API Management で API サブスクリプション キー保護を適用します。
  level: 300
  duration: 60
  islab: true
  primarytopics:
    - Web Application Firewall (WAF)
    - Azure App Service and Function App security
    - Azure API Management
---

# ラボのセットアップ

ラボで使用されるリソースを展開するには、次の手順に従います。

1. **Azure portal ** (`https://portal.azure.com`) を開き、**User1** でサインインします。


1. ポータル検索バーで、「**カスタム テンプレートのデプロイ**」を検索して開きます。

1. **[エディターで独自のテンプレートをビルド]** を選択してから **[ファイルの読み込み]** を選択します。

1. ラボ VM の **F:\AllFiles\Lab-4C** フォルダーから **lab-4c-setup.json** を選択し、次に **[保存]** を選択します。

1. **[基本]** ページで、**[地域]** が `centralus` に設定されていることを確認します。


1. **[確認と作成]** を選択し、次に **[作成]** を選択します。

1. デプロイが **[成功]** と表示されるまで待ってから続行します。

===

# Azure アプリ Services と API Management をセキュリティで保護する

セキュリティ評価により、環境内の複数の Web アプリケーション プラットフォームのギャップが特定されました。

- 受信 Web トラフィックに対して WAF のブロック ポリシーは適用されません。
- App Service および関数アプリ エンドポイントを使用すると、広範なアクセスが可能になります。
- API 呼び出しは、サブスクリプション キーの適用なしで受け入れられます。

このラボでは、検出モードでの WAF の動作の検証、防止モードへの切り替え、Microsoft Entra 認証の適用、ネットワーク制限の適用、APIM でのサブスクリプション キー保護の要求を行います。

このラボでは、次のことを行います。

- WAF 検出モードのログを検証します。
- WAF を検出から防止に切り替え、要求のブロックを確認します。
- App Service の Entra 認証 (Easy Auth) を有効にします。
- App Service と関数アプリへのネットワーク アクセスを制限します。
- API Management でサブスクリプションに必要なアクセスを構成します。
- キーに必要な API の動作を検証します。

この演習の所要時間は約 **60** 分です。

> **注**: このラボでは、固定の Application Gateway `sc500-lab4c-agw` と、名前が `sc500-lab4c-apim-`、`sc500-lab4c-webapp-`、`sc500-lab4c-func-` で始まる生成されたサービスを使用します。 ラボ全体を通じて、`<apim-name>`、`<web-app-name>`、`<function-app-name>` はこれらのリソースを表します。 `sc500-lab4c-rg` リソース グループには、各サービス タイプが正確に 1 つずつ含まれています。

---

## 構成済み状態をレビューする

1. Azure portal で、**[リソース グループ]** を開き、**sc500-lab4c-rg** を選択します。

1. 次のリソースが存在することを確認します。

    - **sc500-lab4c-agw**
    - **<apim-name>**
    - **<web-app-name>**
    - **<function-app-name>**

1. **sc500-lab4c-agw** を開き、アタッチされた WAF ポリシーが現在、**検出**モードであることを確認します。

---

## WAF 検出モードを検証する

1. [Azure portal](https://portal.azure.com) に **User1** アカウントでサインインします。

1. **[アプリケーション ゲートウェイ]** を開き、**sc500-lab4c-agw** を選択します。

1. アタッチされた WAF ポリシーを開き、以下を確認します。

    - モード: **検出**
    - ルール セット: OWASP CRS (現在構成されいているバージョン)

1. Azure portal で **Cloud Shell** を開きます。

1. Application Gateway パブリック エンドポイントと SQL インジェクション スタイルのペイロードを使用して、テスト要求を送信します。

    ```bash
    curl -H "X-Scan-Test: 1" "http://<agw-public-ip>/?id=1+UNION+SELECT+NULL,username,password+FROM+users--"
    ```

1. **[Log Analytics ワークスペース]** を開き、**sc500-lab4c-log** を選択します。

1. 次のようなクエリを実行して、WAF が要求をログに記録したことを確認します。

    ```kusto
    AzureDiagnostics
    | where ResourceType == "APPLICATIONGATEWAYS"
    | where requestUri_s contains "UNION"
    | sort by TimeGenerated desc
    ```

1. 診断データが届くまで最大 **10 分** 待ちます。 要求が表示されるまで 1、2 分ごとにクエリの実行をやり直します。

1. 要求が検出モードでログに記録されていることを確認します。

---

## WAF を防止モードに切り替えて再テストする

1. **sc500-lab4c-agw** の WAF ポリシーに戻ります。

1. モードを **[検出]** から **[防止]** に変更します。

1. ポリシーを保存します。

1. Cloud Shell から同じ `curl` テストをもう一度実行します。

1. 要求がブロックされていることを確認します (通常は HTTP 403)。

1. 結果をノートに記録します。

    | テスト | 予想される結果 |
    |------|-----------------|
    | 検出モードの要求 | ログに記録されましたが、ブロックされていません |
    | 防止モードの要求 | ブロック済み |

---

## App Service 認証を有効にする

1. **[App Services]** を開き、**<web-app-name>** を選択します。

1. **[認証]** を開き、**[ID プロバイダーを追加]** を選択します。

1. **[ID プロバイダー]** で、**[Microsoft]** を選択します。

1. プロバイダーを次のように構成します。

    | 設定 | Value |
    |---------|-------|
    | **テナント構成** | ワークフォース構成 (現在のテナント) |
    | **アプリの登録の種類** | 新しいアプリ登録を作成する |
    | **サポートされているアカウントの種類** | 現在のテナント - 単一テナント |
    | **アクセスの制限** | 認証を必須にする |
    | **認証されていない要求** | HTTP 302 Found リダイレクト |
    | **リダイレクト先** | Microsoft |

1. **[追加]** を選択します。 Microsoft の ID プロバイダーが一覧に表示されていて、認証が有効になっていることを確認します。

1. プロバイダー作成が失敗したり、ページ上でシークレットの欠落が報告されたりした場合は、Cloud Shell を使って同じプロバイダーを設定します。 `<web-app-name>` を生成された App Service 名に置き換えます。

    ```bash
    WEB_APP_NAME='<web-app-name>'
    TENANT_ID=$(az account show --query tenantId -o tsv)
    APP_URL="https://$WEB_APP_NAME.azurewebsites.net"

    APP_ID=$(az ad app create \
      --display-name "$WEB_APP_NAME-auth" \
      --sign-in-audience AzureADMyOrg \
      --web-redirect-uris "$APP_URL/.auth/login/aad/callback" \
      --query appId -o tsv)

    CLIENT_SECRET=$(az ad app credential reset \
      --id "$APP_ID" \
      --append \
      --display-name app-service-auth \
      --query password -o tsv)

    test -n "$APP_ID" || { echo "The app registration could not be created."; exit 1; }
    test -n "$CLIENT_SECRET" || { echo "The client secret could not be created."; exit 1; }

    az webapp config appsettings set \
      --resource-group sc500-lab4c-rg \
      --name "$WEB_APP_NAME" \
      --settings MICROSOFT_PROVIDER_AUTHENTICATION_SECRET="$CLIENT_SECRET" \
      --output none

    SUBSCRIPTION_ID=$(az account show --query id -o tsv)

    AUTH_BODY=$(jq -n \
      --arg clientId "$APP_ID" \
      --arg issuer "https://sts.windows.net/$TENANT_ID/v2.0" \
      '{properties:{platform:{enabled:true,runtimeVersion:"~1"},globalValidation:{requireAuthentication:true,unauthenticatedClientAction:"RedirectToLoginPage",redirectToProvider:"azureactivedirectory"},identityProviders:{azureActiveDirectory:{enabled:true,registration:{openIdIssuer:$issuer,clientId:$clientId,clientSecretSettingName:"MICROSOFT_PROVIDER_AUTHENTICATION_SECRET"}}},login:{tokenStore:{enabled:true}}}}')

    az rest --method put \
      --uri "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/sc500-lab4c-rg/providers/Microsoft.Web/sites/$WEB_APP_NAME/config/authsettingsV2?api-version=2022-03-01" \
      --body "$AUTH_BODY" \
      --output none

    az rest --method get \
      --uri "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/sc500-lab4c-rg/providers/Microsoft.Web/sites/$WEB_APP_NAME/config/authsettingsV2?api-version=2022-03-01" \
      --query '{Enabled:properties.platform.enabled,UnauthenticatedAction:properties.globalValidation.unauthenticatedClientAction}' \
      --output table

    unset CLIENT_SECRET AUTH_BODY
    ```

1. 認証設定が伝達されるまで、最大 **2 分** 待ちます。 プライベート ブラウザー ウィンドウでアプリの URL を開き、Microsoft サインインにリダイレクトされることを確認します。

---

## App Service と関数アプリにネットワーク制限を適用する

1. **<web-app-name>** で、**[ネットワーク]**、**[アクセス制限]** の順に開きます。

1. Application Gateway に関連付けられている承認済みサブネットの許可ルールを追加します。

1. 一致しないトラフィックに対して、既定のアクションを **[拒否]** に設定します。

1. [変更の保存]。

1. **Function Apps** を開き、**<function-app-name>** を選択します。

1. **[ネットワーク]** を開き、アクセス制限を構成します。

1. 承認された関数サブネットのみに許可ルールを追加します。

1. 既定のアクションを **[拒否]** に設定します。

1. [変更の保存]。

---

## API Management でサブスクリプション キー保護を適用する

1. **[API Management サービス]** を開き、**<apim-name>** を選択します。

1. **[API]** を開き、事前構成済みの API を選択します。

1. [API 設定] で、**[サブスクリプションが必要]** を **[必須]** に設定します。

1. [変更の保存]。

1. テスト サブスクリプションを作成または開き、キーをコピーします。

1. 鍵を使用してテストします。 `Ocp-Apim-Subscription-Key` ヘッダーを含め、モック API から HTTP **200** が返されることを確認します。

1. 鍵を使用せずにテストします。 サブスクリプション キーのヘッダーを削除し、APIM から HTTP **401** が返されることを確認します。

1. ノートに結果を記録します。

    | 要求の種類 | 予想される結果 |
    |--------------|-----------------|
    | With key | HTTP 200 |
    | キーなし | HTTP 401 |

---

## まとめ

このラボでは、Web および API ワークロード用の階層化されたコントロールを実装しました。

- WAF 検査と防止モードでのアクティブなブロック。
- App Service に対する Entra 認証を使用した ID の適用。
- App Service と関数アプリのネットワーク絞り込み。
- APIM サブスクリプション キーによる API 受付制御。

これらのコントロールでは、悪用可能性を減らし、認証されていないアクセス パスを制限し、ネットワーク層とアプリケーション層の両方でポリシーを適用します。
