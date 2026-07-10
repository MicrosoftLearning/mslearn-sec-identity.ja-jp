# Microsoft Entra ユーザー ID を持つ Foundry エージェントを作成する

このガイドでは、Microsoft Foundry でエージェントを作成し、自動でプロビジョニングされる Entra ID オブジェクトを特定して、エージェントにユーザー アカウント作成のブループリント権限を付与したうえで、そのユーザー アカウントを作成する方法を説明します。 これはラボ演習ではなく、環境のセットアップや自主的な練習のためのリファレンスとして利用できる、完全なエンドツーエンドの手順です。

**この重要性について:** Foundry でエージェントを作成すると、Entra ID に "エージェント ID" (サービス プリンシパル) が自動的にプロビジョニングされます。** しかし、エージェントの "ユーザー アカウント" は自動的には作成されません。これは、[すべてのユーザー] に標準ユーザーと共に表示される 2 番目の Entra ID オブジェクトです。****** そのオブジェクトは明示的に作成する必要があり、そのための権限を最初にブループリントに付与しておく必要があります。

---

## 前提条件

| タスク | 必要なロール |
|------|--------------|
| Foundry プロジェクトとエージェントを作成する | サブスクリプション スコープでの **Foundry アカウント オーナー**またはサブスクリプション レベルでの**共同作成者/オーナー** |
| 既存のプロジェクトでエージェントを使用して作業する | プロジェクト スコープでの **Foundry ユーザー** |
| Entra ID 内でエージェント ID とブループリントを表示する | 任意の Microsoft Entra ユーザー アカウント (閲覧には管理者ロール不要) |
| ブループリントに権限を付与する | **特権ロール管理者** (アプリケーションのアクセス許可) |
| エージェントのユーザー アカウントを作成する | **エージェント ID 管理者**または**ユーザー管理者** |

また、権限付与の手順を行うには、2 つの PowerShell モジュールがインストールされている必要があります。 **PowerShell 7.x** ターミナルで以下を実行します (Windows PowerShell 5.x はどちらのモジュールでもサポートされていません)。

```powershell
Install-Module Microsoft.Entra.Authentication -Scope CurrentUser
Install-Module Microsoft.Entra.Applications -Scope CurrentUser
```

- **`Microsoft.Entra.Authentication`** — Entra テナントとのセッション確立に使われる `Connect-Entra` 認証コマンドレットを提供します。
- **`Microsoft.Entra.Applications`** (v1.3 以降) — パート 3 でブループリントへの権限付与に使用する `Add-EntraPermissionToCreateAgentUsersToAgentIdentityBlueprintPrincipal` を提供します。

> **注:** `Microsoft.Entra` アンブレラ パッケージではなく、ターゲットを絞ったこれら 2 つのサブモジュールをインストールします。 アンブレラはすべてのサブモジュール (`Microsoft.Entra.SignIns` を含む) を依存関係としてインストールしようと試みますが、PSGallery の可用性の問題により失敗することがあります。 このガイドの手順には、上記 2 つのモジュールだけで十分です。

---

## パート 1 — Microsoft Foundry でエージェントを作成する

> **出典:** [Microsoft Foundry クイックスタート](https://learn.microsoft.com/azure/foundry/quickstarts/get-started-code)、[Foundry ポータルでエージェントを作成する](https://learn.microsoft.com/azure/app-service/tutorial-ai-integrate-azure-ai-agent-node#create-an-agent-in-microsoft-foundry)

1. [Microsoft Foundry ポータル](https://ai.azure.com)にサインインします。

1. ポータルの右上の領域で **[新しい Foundry]** のトグルがアクティブになっていることを確認します。 これらの手順では現在の (クラシックではない) Foundry エクスペリエンスを使用します。

1. 既存のプロジェクトがない場合は、次のようにして作成します。
   - 右上のメニューから **[新しい Foundry]** を選択します。
   - **[新しいプロジェクトの作成]** を選択し、プロジェクト名を入力して **[作成]** を選択します。

1. プロジェクトのホーム ページから **[ビルドの開始]** を選択し、次に **[エージェントの作成]** を選択します。

1. エージェントの名前を入力し、**[作成]** を選択します。 プロビジョニングが完了すると、エージェントのプレイグラウンドが開きます。

    > **注:** プロジェクトで最初のエージェントが作成されると、Foundry は Microsoft Entra ID に 2 つのオブジェクトを自動的にプロビジョニングします: **エージェント ID ブループリント**と**共有プロジェクト エージェント ID** です。 これらのオブジェクトはすぐに Entra 管理センターに表示されます。 このプロビジョニングを行うために、追加の設定は必要ありません。

1. オプションでエージェントへの指示 (例: "あなたは頼りになるアシスタントです") を追加し、プレイグラウンドでテストすることもできます。** この手順は ID プロビジョニングに必須ではありません。

1. **[保存]** を選択します。

---

## パート 2 — Entra ID でブループリントとエージェントの ID を特定する

> **出典:** [エージェント ID の表示とフィルター処理](https://learn.microsoft.com/entra/agent-id/agent-lists)、[Foundry のエージェント ID の概念](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity#foundry-integration)

### ブループリントのアプリケーション ID を見つける (PowerShell に必要)

ブループリントのアプリケーション ID は、Entra 管理センターまたは Azure portal のどちらからでも取得できます。

**オプション A — Entra 管理センター:**

1. [Microsoft Entra 管理センター](https://entra.microsoft.com)にサインインします。

1. **[Entra ID]** > **[エージェント]** > **[エージェント ブループリント]** に移動します。

1. 一覧には、**プロジェクト ブループリント** (Foundry プロジェクトに基づく名前。例: *myproject*) と、1 つ以上の**エージェント ブループリント** (それぞれの個別エージェントに基づく名前。例: *myagent*) が含まれています。 パート 1 で作成した特定のエージェントに基づいた名前の**エージェント ブループリント**を選択します。

   > **注:** パート 3 で付与する権限はエージェント ブループリントに適用され、プロジェクト ブループリントには適用されません。 プロジェクト ブループリントに付与しても、エージェントのユーザー アカウント作成には影響しません。

1. ブループリントの詳細ページで、**ブループリントのアプリケーション ID** をコピーします。 この値を保存します。これはパート 3 で必要です。

**オプション B — Azure portal の JSON ビュー:**

1. [Azure portal](https://portal.azure.com) にサインインし、Foundry プロジェクト リソースに移動します。

1. **[概要]** ペインで **[JSON ビュー]** を選択し、最新の API バージョンを選択します。

1. 結果の JSON 出力から、`agentIdentityBlueprintId` 値を見つけてコピーします。

### エージェント ID を確認する

1. Entra 管理センターで、**[Entra ID]** > **[エージェント]** > **[エージェント ID]** に移動します。

1. Foundry プロジェクトのエージェント ID を見つけます。 それを選択し、詳細ペインを確認します。

1. エージェント ID の**オブジェクト ID** をメモします。 これは、ユーザー アカウントがリンクされる親 ID になります。

---

## パート 3 — ユーザー アカウント作成のブループリント権限を付与する

> **出典:** [Add-EntraPermissionToCreateAgentUsersToAgentIdentityBlueprintPrincipal](https://learn.microsoft.com/powershell/module/microsoft.entra.applications/add-entrapermissiontocreateagentuserstoagentidentityblueprintprincipal?view=entra-powershell)

既定では、エージェント ID ブループリントにエージェント ユーザー アカウントの作成権限は**ありません**。 付与する必要がある特定の権限は、`AgentIdUser.ReadWrite.IdentityParentedBy` (アプリ ロール ID: `4aa6e624-eee0-40ab-bdd8-f9639038a614`) です。

この手順では、PowerShell を使用してその権限をブループリントのサービス プリンシパルに割り当てます。

1. PowerShell セッションを開き、必要なスコープでテナントに接続します。 `<your-tenant-id>` を実際のテナント ID に置き換えます。

    ```powershell
    Connect-Entra -TenantId "<your-tenant-id>" `
        -Scopes 'Application.Read.All', `
                'AppRoleAssignment.ReadWrite.All', `
                'AgentIdentityBlueprint.UpdateAuthProperties.All', `
                'AgentIdUser.ReadWrite.IdentityParentedBy'
    ```

    > **注:** 4 つのスコープすべてが必要です。 `Application.Read.All` により、コマンドレットはブループリントのサービス プリンシパルを検索できます。 `AppRoleAssignment.ReadWrite.All` により、実際の付与を行う `/appRoleAssignments` への POST が可能となります。委任認証では、サインインしているアカウントが特権ロール管理者であっても、このスコープがトークンに明示的に含まれている必要があります。 `AgentIdentityBlueprint` と `AgentIdUser` の 2 つのスコープは、付与対象のエージェント固有の権限です。

1. パート 2 でコピーしたアプリケーション ID を使用して、ブループリントに権限を付与します。

    ```powershell
    Add-EntraPermissionToCreateAgentUsersToAgentIdentityBlueprintPrincipal `
        -AgentBlueprintId "<blueprint-application-id>"
    ```

1. 出力に以下の値が含まれていることを確認します。

    | 出力のフィールド | 必要な値 |
    |-------------|----------------|
    | `PermissionName` | `AgentIdUser.ReadWrite.IdentityParentedBy` |
    | `appRoleId` | `4aa6e624-eee0-40ab-bdd8-f9639038a614` |
    | `AgentBlueprintId` | 指定したブループリントのアプリケーション ID |

    > **注:** このコマンドレットは、ブループリントのサービス プリンシパル (Foundry が自動的に作成する) がテナント内に既に存在していることを前提としています。 コマンドレットが not-found エラーで失敗した場合は、再試行する前に **[Entra ID]** > **[エージェント]** > **[エージェント ブループリント]** にブループリントが表示されていることを確認します。

---

## パート 4 — エージェントのユーザー アカウントを作成する

> **出典:** [Microsoft Entra エージェント ID のエージェントのユーザー アカウント](https://learn.microsoft.com/entra/agent-id/agent-users)、[エージェント ID はどのように作成されますか?](https://learn.microsoft.com/entra/agent-id/agent-id-creation-channels)

エージェントのユーザー アカウントは、**[すべてのユーザー]** に表示されるオブジェクトであり、エージェントがユーザー ID として機能することを可能にします。メールボックス、Teams、カレンダー、その他のユーザータイプ トークン (`idtyp=user`) を必要とするリソースへのアクセスが可能になります。

このオブジェクトを作成するには、**エージェント ID 管理者**または**ユーザー管理者**のロールが必要です。

### Microsoft Graph API を使用して作成する

Entra 管理センターのエージェント ID 詳細ページ (**[Entra ID]** > **[エージェント]** > **[エージェント ID]** > エージェントを選択) には、概要、カスタム セキュリティ属性、所有者とスポンサー、付与されたアクセス許可、監査ログ、サインイン ログのみが表示されます。 現在、ポータルの UI には **[ユーザー アカウントの作成]** オプションはありません。 代わりに [Graph エクスプローラー](https://developer.microsoft.com/graph/graph-explorer)または PowerShell を使用します。

1. [Graph エクスプローラー](https://developer.microsoft.com/graph/graph-explorer)を開き、**エージェント ID 管理者**または**ユーザー管理者**であるアカウントでサインインします。

1. クエリを実行する前に、必要な委任されたアクセス許可を Graph エクスプローラーに付与します。 クエリ ウィンドウ上部の **[アクセス許可の修正]** を選択し、以下の各項目を検索して、それぞれに対して **[同意]** を選択します。
   - `AgentIdUser.ReadWrite.IdentityParentedBy`
   - `AgentIdUser.ReadWrite.All`(最初のものが不十分な場合の代替としての同意)**

   > **注:** Graph エクスプローラーは、Entra のロール割り当てとは独立して独自の同意済みアクセス許可セットを維持しています。 エージェント ID 管理者のロールがアクティブなグローバル管理者であっても、Graph エクスプローラーが `AgentIdUser` スコープに対して明示的に同意されていない場合は、403 エラーが表示されます。 **[アクセス許可の修正]** ボタンはクエリ ウィンドウの上部にあり、左側のナビゲーションにはありません。

1. Entra 管理センターでエージェント ID (**[Entra ID]** > **[エージェント]** > **[エージェント ID]** > エージェントを選択) を開き、[概要] ペインから**オブジェクト ID** をコピーします。

1. Graph エクスプローラーでメソッドを **[POST]** に、URL を次のように設定します。

    ```
    https://graph.microsoft.com/v1.0/users/microsoft.graph.agentUser
    ```

1. **[要求本文]** タブに以下を入力します。 プレースホルダーの値を、エージェントの詳細と認証済みドメインに置き換えます。

    ```json
    {
      "accountEnabled": true,
      "displayName": "<your-agent-display-name>",
      "mailNickname": "<mailnickname-no-spaces>",
      "userPrincipalName": "<mailnickname>@<yourdomain>.com",
      "identityParentId": "<agent-identity-object-id>"
    }
    ```

    | フィールド | メモ |
    |-------|-------|
    | `accountEnabled` | `true` に設定 |
    | `displayName` | [すべてのユーザー] に表示される、人間が判読できる名前 |
    | `mailNickname` | エイリアス —スペースまたは特殊文字は使用できません |
    | `userPrincipalName` | テナント内の**検証済み**ドメインを使用する必要があります — 正確なドメイン文字列は、**[Entra ID]** > **[カスタム ドメイン名]** で確認してください |
    | `identityParentId` | パート 2 のエージェント ID の**オブジェクト ID** (アプリケーション ID ではありません) |

    > **注:** 5 つのフィールドすべてが必要です。 `identityParentId` には、エージェント ID の**オブジェクト ID** を指定します。パート 3 で使用したブループリントのアプリケーション ID ではありません。

1. **[クエリの実行]** を選択します。 応答が成功すると、HTTP **201 作成済み**が返され、応答本文には新しいエージェント ユーザー オブジェクトが加わります。 応答で以下のフィールドを確認します。

    | 応答フィールド | 必要な値 |
    |---------------|----------------|
    | `identityParentId` | パート 2 のエージェント ID のオブジェクト ID |
    | `agentIdentityBlueprintId` | パート 3 のブループリントのアプリケーション ID |
    | `passwordProfile` | `null` — パスワード認証情報が存在しません |
    | `userType` | `Member` |

    応答に `identityParentId` と `agentIdentityBlueprintId` の両方が存在していれば、完全なブループリント → [エージェント ID] → [ユーザー アカウント階層] がリンクされていることを確認できます。

---

## パート 5 — Entra ID でユーザー アカウントを確認する

> **出典:** [組織内のエージェント ID を管理する](https://learn.microsoft.com/entra/agent-id/manage-agent-identities-admin)

1. Entra 管理センターで、**[Entra ID]** > **[ユーザー]** > **[すべてのユーザー]** に移動します。

1. エージェントの表示名を検索します。 標準のユーザー アカウントと一緒にエージェント ユーザー アカウントが表示されるはずです。

1. ユーザー アカウント エントリを選択し、次のことを確認します。

    - アカウント タイプが (標準ユーザーやゲストではなく) エージェント ユーザーとして表示されている
    - アカウントにパスワードまたはパスキーの認証情報がない - 親エージェント ID 経由でのみ認証される
    - アカウントに特権管理者ロールを割り当てることはできない

1. **[Entra ID]** > **[エージェント]** > **[エージェント ID]** に戻り、エージェント ID を選択し、詳細ページの上部近くに **"このエージェント ID には関連付けられたエージェント ユーザーが存在します"** と示された灰色のバナーが表示されていることを確認します。 **[表示]** を選択し、パート 4 で作成したユーザー アカウントへリンクされていることを確認します。

    > **注:** このリンクの確認は、[概要] ペイン内の専用プロパティやセクションではなく、バナーとして表示されます。 バナーが表示されない場合は、エージェント ID 詳細ページの一番上までスクロールしてください。

---

## オブジェクト モデルの概要

すべてのパートを完了すると、エージェントには 3 つの異なる Entra ID オブジェクトが存在します。

| Object | Type | Entra 管理センターでの場所 |
|--------|------|-------------------------------|
| エージェント ID のブループリント | アプリケーション登録 (タイプ指定) | [エージェント] > [エージェント ブループリント] |
| エージェント ID | サービス プリンシパル (タイプ指定) | [エージェント] > [エージェント ID] |
| エージェントのユーザー アカウント | ユーザー (タイプ指定、パスワードなし) | [ユーザー] > [すべてのユーザー] |

これら 3 つのオブジェクトには固定された階層があります: [ブループリント] → [エージェント ID] → [ユーザー アカウント]。 それぞれの関係は 1 対 1 であり、作成後に変更することはできません。 ユーザー アカウントは、親エージェント ID に発行されたトークンを提示することでのみ認証できます。独立した認証情報は持っていません。

---

## リファレンス ドキュメント

- [Microsoft Foundry のエージェント ID の概念](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity)
- [エージェント ID ブループリントを作成する](https://learn.microsoft.com/entra/agent-id/create-blueprint)
- [エージェント ID を作成する](https://learn.microsoft.com/entra/agent-id/create-delete-agent-identities)
- [エージェントのユーザー アカウント](https://learn.microsoft.com/entra/agent-id/agent-users)
- [組織内のエージェント ID を管理する](https://learn.microsoft.com/entra/agent-id/manage-agent-identities-admin)
- [Add-EntraPermissionToCreateAgentUsersToAgentIdentityBlueprintPrincipal](https://learn.microsoft.com/powershell/module/microsoft.entra.applications/add-entrapermissiontocreateagentuserstoagentidentityblueprintprincipal?view=entra-powershell)
