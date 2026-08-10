---
lab:
  title: AKS と Defender for Containers を使用してコンテナー ワークロードをセキュリティ保護する
  description: 事前にプロビジョニングされた AKS クラスターで Defender for Containers を有効にし、コンテナーとレジストリの監視を確認し、Azure Container Registry でのアクセスとネットワーク セキュリティのギャップを修復します。
  level: 300
  duration: 45
  islab: true
  primarytopics:
    - Microsoft Defender for Containers
    - Azure Kubernetes Service (AKS)
    - Azure Container Registry (ACR)
---

# ラボのセットアップ

次の手順を完了してから、ラボで使用するリソースを展開し、コンテナー イメージをシードする演習を開始します。

1. **Azure portal ** (`https://portal.azure.com`) を開き、**User1** でサインインします。

1. ポータルの上部バーで、**[Cloud Shell]** アイコン (**>_**) を選択します。 ダイアログが表示されたら **[Bash]** と **[ストレージ アカウントは必要ありません]** を選択します。

1. Defender プランに必要な Microsoft.Security リソース プロバイダーを登録し、登録が完了するまで待ちます。

    ```bash
    az provider register --namespace Microsoft.Security --wait
    az provider show --namespace Microsoft.Security --query registrationState -o tsv
    ```

    出力が `Registered` であることを確認してから続行します。

1. **User2** と **User3** の資格情報を使い、各アカウントの正確なユーザー名をコピーします。 ロール割り当ての手順で両方のユーザー名を使用できるようにしておきます。

1. Cloud Shell で `<User2-UPN>` と `<User3-UPN>` を正確なユーザー名に置き換え、両方のディレクトリ オブジェクト ID を解決します。

    ```bash
    USER2_UPN='<User2-UPN>'
    USER3_UPN='<User3-UPN>'

    USER2_OBJECT_ID=$(az ad user show --id "$USER2_UPN" --query id -o tsv)
    USER3_OBJECT_ID=$(az ad user show --id "$USER3_UPN" --query id -o tsv)

    test -n "$USER2_OBJECT_ID" || { echo "User2 could not be resolved in Microsoft Entra ID."; exit 1; }
    test -n "$USER3_OBJECT_ID" || { echo "User3 could not be resolved in Microsoft Entra ID."; exit 1; }

    echo "User2 object ID: $USER2_OBJECT_ID"
    echo "User3 object ID: $USER3_OBJECT_ID"
    ```

1. 両方のオブジェクト ID をコピーします。 ポータル メンバー ピッカーがどちらのアカウントも返さない場合に備え、それらを使用できるようにしておきます。


1. ポータル検索バーで、「**カスタム テンプレートのデプロイ**」を検索して開きます。

1. **[エディターで独自のテンプレートをビルド]** を選択してから **[ファイルの読み込み]** を選択します。

1. ラボ VM の **F:\AllFiles\Lab-4B** フォルダーから **lab-4b-setup.json** を選択し、次に **[保存]** を選択します。


1. **[確認と作成]** を選択し、次に **[作成]** を選択します。

    > **注**: AKS のデプロイには、通常 5 から 10 分かかります。 デプロイが **[成功]** と表示されるまで待ってから続行します。

1. Cloud Shell をもう一度開き、次のコマンドを実行して、生成されたレジストリ名を特定し、必要なイメージをインポートして、タグを確認します。

    ```bash
    ACR_NAME=$(az acr list --resource-group sc500-lab4b-rg --query "[0].name" -o tsv)
    echo $ACR_NAME
    test -n "$ACR_NAME" || { echo "No container registry was found in sc500-lab4b-rg."; exit 1; }
    az acr import --name $ACR_NAME --source docker.io/library/nginx:1.19.0-alpine --image nginx:1.19.0-alpine --force
    az acr repository show-tags --name $ACR_NAME --repository nginx --output table
    ```

1. 出力に `1.19.0-alpine` が含まれていることを確認し、表示されたレジストリ名をコピーしてから Cloud Shell を閉じます。

===

# AKS と Defender for Containers を使用してコンテナー ワークロードをセキュリティ保護する

あなたの組織は AI 推論マイクロサービスを AKS で実行しています。 セキュリティ レビューで、次の 3 つの高リスク ギャップが特定されました。

- ランタイム脅威検出がクラスターに対して有効化されていません。
- コンテナー レジストリの既知の脆弱性を見つけるためのスキャンが行われていません。
- レジストリへのアクセス許可が過剰に付与されており、ネットワーク境界による制限もありません。

このラボでは、Defender for Containers の有効化、決定論的監視設定の確認、スキャン用の ACR イメージの準備、ID およびネットワーク制御の強化を通じてこれらのギャップを修復します。

このラボでは、次のことを行います。

- Defender for Containers をサブスクリプションに対して有効にします。
- Defender for Cloud のコンテナーおよびレジストリの監視設定を確認します。
- ACR イメージをインポートし、脆弱性スキャンが可能であることを確認します。
- ACR 管理者ユーザーを無効にして、スコープを限定した RBAC ロールを割り当てます。
- ACR ネットワーク アクセスを、承認済みのパブリック ネットワーク範囲に限定します。

この演習の所要時間は約 **45** 分です。

> **注**: このラボでは、事前プロビジョニングされた AKS クラスター `sc500-lab4b-aks`、仮想ネットワーク `sc500-lab4b-vnet`、名前が `sc500lab4bacr` で始まるコンテナー レジストリを使用します。 ラボ全体を通じて、`<acr-name>` は生成されたレジストリ名を表します。

---

## 事前構成済みの状態をレビューする

1. Azure portal で **[リソース グループ]** を開いて **sc500-lab4b-rg** を選択します。

1. リソース グループに次のものが含まれていることを確認します。

    - **sc500-lab4b-aks**
    - **sc500-lab4b-vnet**
    - 名前が **sc500lab4bacr** で始まる 1 つのコンテナー レジストリ

1. **<acr-name>** を開き、**[サービス]** > **[リポジトリ]** を選択し、次に **[nginx]** を選択します。

1. **[1.19.0-alpine]** タグが存在していることを確認します。

---

## Defender for Containers を有効にする

1. Azure portal で **Microsoft Defender for Cloud** を開きます。

1. **[管理]** を展開し、**[環境設定]** を選択し、**[すべてを展開]** を選択します。

1. アクティブなラボ サブスクリプションを選択します。

1. **[設定]、[Defender プラン]** で、**[コンテナー]** を **[オン]** に設定して、**[保存]** を選択します。

1. 成功通知が表示され、**[コンテナー]** 行の **[監視カバレッジ]** の下に **[完全]** と表示されていることを確認します。

---

## コンテナーとレジストリ監視の検証

1. **[コンテナー]** 行で、リソースの数量に 1 つのコンテナー レジストリと、**sc500-lab4b-aks** からの Kubernetes コアが含まれていることを確認します。

1. **[コンテナー]** プランの **[設定 >]** を選択します。

1. **[設定と監視]** で、**[レジストリ アクセス]** が **[オン]** に設定され、その構成に **[セキュリティの結果: オン]** と表示されていることを確認します。

1. **[続行]** を選択して Defender プランのページに戻り、ボタンが有効になっている場合は、**[保存]** を選択します。

1. **<acr-name>** に戻り、**[サービス]** > **[リポジトリ]** > **[nginx]** の順に選択し、**[1.19.0-alpine]** タグを選択します。

1. イメージ メタデータが正常に開くことを確認します。

> [!NOTE]
> Defender の推奨事項やイメージ脆弱性の検出結果は非同期に生成され、新規サブスクリプションに表示されるまでに最大 24 時間かかることがあります。 これらは、このラボを完了させるうえで必要ありません。 結果が利用可能な場合は、必要に応じて、**[Microsoft Defender for Cloud]** > **[推奨事項]** から **sc500-lab4b-aks** または **<acr-name>** でフィルター処理すると確認できます。

---

## ACR アクセス制御を適用する

1. **<acr-name>** で **[設定]** > **[アクセス キー]** を開きます。

1. **[管理者ユーザー]** を **[無効]** に設定するために、そのチェックボックスを**オフ**にします。

1. **<acr-name>** の **[アクセス制御 (IAM)]** を開きます。

1. **[+ 追加]** > **[ロールの割り当ての追加]** の順に選択します。

1. **[ロール]** タブで **[AcrPull]** を選択してから **[次へ]** を選択します。

1. **[メンバー]** タブで、**[ユーザー、グループ、またはサービス プリンシパル]** を選択してから、**[+ メンバーの選択]** を選択します。

1. 先ほどコピーした正確な **User2** ユーザー名を貼り付けます。 マッチするアカウントが表示されたら、それを選択し、**[レビューと割り当て]** を選択します。

1. このロール割り当てのプロセスを繰り返します。**[+ 追加]** > **[ロールの割り当ての追加]** を選択し、**[AcrPush]** を選択して **[次へ]** を選択します。

1. **[メンバー]** タブで、**[ユーザー、グループ、またはサービス プリンシパル]** を選択してから、**[+ メンバーの選択]** を選択します。

1. 先ほどコピーした正確な **User3** ユーザー名を貼り付けます。 マッチするアカウントが表示されたら、それを選択し、**[レビューと割り当て]** を選択します。

1. どちらのメンバー ピッカーも一致するアカウントを返さない場合は、Cloud Shell を開き、次のフォールバックを実行します。 `<acr-name>`、`<User2-object-id>`、`<User3-object-id>` を、先程コピーした値に置き換えます。 コマンドでは、不足している割り当てのみが作成されます。

    ```bash
    ACR_NAME='<acr-name>'
    USER2_OBJECT_ID='<User2-object-id>'
    USER3_OBJECT_ID='<User3-object-id>'
    ACR_ID=$(az acr show --name "$ACR_NAME" --query id -o tsv)
    test -n "$ACR_ID" || { echo "The container registry could not be resolved."; exit 1; }

    USER2_ASSIGNMENT_COUNT=$(az role assignment list \
      --scope "$ACR_ID" \
      --query "[?principalId=='$USER2_OBJECT_ID' && roleDefinitionName=='AcrPull'] | length(@)" \
      -o tsv)

    if [ "$USER2_ASSIGNMENT_COUNT" = "0" ]; then
      az role assignment create \
        --assignee-object-id "$USER2_OBJECT_ID" \
        --assignee-principal-type User \
        --role AcrPull \
        --scope "$ACR_ID"
    fi

    USER3_ASSIGNMENT_COUNT=$(az role assignment list \
      --scope "$ACR_ID" \
      --query "[?principalId=='$USER3_OBJECT_ID' && roleDefinitionName=='AcrPush'] | length(@)" \
      -o tsv)

    if [ "$USER3_ASSIGNMENT_COUNT" = "0" ]; then
      az role assignment create \
        --assignee-object-id "$USER3_OBJECT_ID" \
        --assignee-principal-type User \
        --role AcrPush \
        --scope "$ACR_ID"
    fi

    az role assignment list \
      --scope "$ACR_ID" \
      --query "[?roleDefinitionName=='AcrPull' || roleDefinitionName=='AcrPush'].{Role:roleDefinitionName,Principal:principalName}" \
      -o table
    ```

1. **[アクセス制御 (IAM)]** > **[ロールの割り当て]** に戻り、**[更新]** を選択して両方の割り当てを確認します:

    - 正確な User2 アカウントは **AcrPull**。
    - 正確な User3 アカウントは **AcrPush**。

---

## ACR ネットワーク アクセスを制限する

1. **<acr-name>** で、**[設定]** > **[ネットワーク]** の順に開きます。

1. **[パブリック アクセス]** タブの **[パブリック ネットワーク アクセス]** で、**[選択されたネットワーク]** を選択します。

1. **[ファイアウォール]** の下で **[クライアント IP アドレスを追加]** を選択します。これは現在のセッションの接続を維持するためです。

1. **[保存]** を選択します。

1. **[選択されたネットワーク]** が保存の後も選択されたままであることを確認します。

1. 構成済みのファイアウォール IP エントリだけが **[ファイアウォール]** の下に表示されていることを確認します。

1. **[選択されたネットワーク]** が有効のときは未承認のパブリック ネットワークからのアクセスが意図的に拒否されることを確認します。

> [!NOTE]
> 現在の Azure portal では、ACR に関して、**[パブリック アクセス]** で **[選択されたネットワーク]** が選択されたときにファイアウォール IP ルールが使用されます。 仮想ネットワークとサブネットの選択を設定するには **[プライベート アクセス]** でプライベート エンドポイント接続を作成することになりますが、このラボでは必須ではありません。

---

## まとめ

このラボでは、Defender for Containers を有効にし、コンテナーとレジストリの監視カバレッジを検証し、脆弱性スキャン用のレジストリ イメージを作成し、レジストリ内の ID とネットワークの露出を修復しました。

これで、コンテナー ワークロードに対するセキュリティ態勢は次のとおりに多層化されました。

- 実行時の状態と構成を Defender for Containers で可視化する。
- Defender のインジェスト後に脆弱性の可視化のために構成されたレジストリ イメージの監視。
- レジストリへの最小特権アクセスを RBAC で付与する。
- 外部への露出をネットワーク制限で縮小する。
