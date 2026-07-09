---
lab:
  title: Privileged Identity Management の構成
  description: PIM 対応のロールの割り当て、アクティブ化設定、承認ワークフローを構成し、Just-In-Time の特権アクセスを適用し、Azure リソースでマネージド ID を有効にします。
  level: 300
  duration: 45
  islab: true
  primarytopics:
    - Microsoft Entra Privileged Identity Management
    - Conditional Access
    - Managed Identity
---

# ラボのセットアップ

ラボ プロファイル - https://labondemand.com/LabProfile/217878

このラボは特別な構成が不要な M365 テナント上で動作します。

===

# Privileged Identity Management の構成

Privileged Identity Management (PIM) は、Azure および Microsoft Entra のロールに対して Just-In-Time (JIT) 特権アクセスを可能にする Microsoft Entra ID サービスです。 永続的な攻撃面が発生する常任管理者アクセスを許可する代わりに、PIM は承認と理由の要件は任意で、ユーザーに限られた期間の上位アクセスの要求とアクティブ化を求めます。

このラボでは、条件付きアクセス管理者ロールのための PIM を構成し、承認ベースのアクティブ化ワークフローを適用し、上位アクセスが期待どおりに動作するかを検証してから、Azure App Service 上でシステム割り当てマネージド ID を有効化します。

このラボでは、次のことを行います。

- 条件付きアクセス管理者ロールを PIM 対応のロールの割り当てとして割り当てる
- 制限時間、理由の要件、承認者を含めてアクティブ化設定を構成する
- 2 つの別々のアカウントを使用してロールのアクティブ化を要求して承認する
- アクティブ化されたロールによって期待されたアクセスが許可されることを確認する
- 事前にプロビジョニングされた App Service でシステム割り当てマネージド ID を有効にする
- ロールを非アクティブ化して Just-In-Time アクセス期間を終了する

この演習の所要時間は約 **45** 分です。

---

## PIM 対応のロールを割り当てる

このセクションでは、条件付きアクセス管理者ロールを適格な割り当てとして **sc500-user01** に割り当てます。 適格な割り当ては、ユーザーがそのロールを永続的に保持するわけではなく、必要なたびに要求してアクティブ化しなければならないことを意味します。

1. `https://entra.microsoft.com` で **[管理者]** の資格情報を使用して Microsoft Entra 管理センターにサインインします。

1. 左側のナビゲーションで、**[ID ガバナンス]** を展開し、**[Privileged Identity Management]** を選択します。

1. **[管理]** で **[Microsoft Entra ロール]** を選択します。

1. **[割り当て]** を選択してから、**[割り当ての追加]** を選択します。

1. **[+ 割り当ての追加]** ページで、次のように構成します。

    | 設定 | Value |
    |---------|-------|
    | **ロールの選択** | 条件付きアクセス管理者 |
    | **[メンバーの選択]** | Adele Vance |
    | **[割り当ての種類]** | [適格] ([次へ] ボタンの使用後) |

1. **[次へ]** を選択してから、**[割り当て]** を選択して割り当てを保存します。

1. **[割り当て]** ページで、**[Adele Vance]** が **[対象の割り当て]** タブの下に **[条件付きアクセス管理者]** のロールで表示されていることを確認します。

    > **注**: [対象の割り当て] はアクセスを許可するものではなく、ユーザーがアクティブ化を要求できるだけです。 現時点で有効なアクセスはありません。

---

## アクティブ化設定を構成する

PIM のロール設定によって、アクティブ化の期間、理由が必要かどうか、承認者が要求ごとに承認しなければならないかなど、アクティブ化プロセスの動作が制御されます。 これから条件付きアクセス管理者のロール設定を構成します。

1. **[Privileged Identity Management] > [Microsoft Entra ロール]** で、**[設定]** を選択します。

1. ロール リストから **[条件付きアクセス管理者]** を検索して選択します。

1. **[編集]** を選択してロール設定を開きます。

1. **[アクティブ化]** タブで、次の設定を構成します。

    | 設定 | Value |
    |---------|-------|
    | **アクティブ化の最大期間** | 1 時間 |
    | **[アクティブ化で必要]** | 妥当性 |
    | **アクティブ化に承認を要求する** | Enabled |
    | **その他の設定** | 既定値のままにする |

1. **[承認者の選択]** で、**[+ メンバーの選択]** を選択します。

1. **[MOD 管理者]** を検索して選択し、**[選択]** を選択します。

1. **[更新]** を選択して、ロール設定を保存します。

1. ロール設定のページに次の内容が表示されたことを確認します。
    - [最大アクティブ化期間]: **[1 時間]**
    - [承認が必要]: **[はい]**
    - [承認者]: **[sc500-approver]**

---

## ロールのアクティブ化要求

ここで、**[sc500-user01]** としてサインインし、ロールのアクティブ化要求を送信します。 これで、特定のタスクを実行するために一時的な上位アクセスが必要なユーザーがシミュレートされます。

1. **InPrivate** または**プライベート** ブラウザー ウィンドウを開きます。

1. `https://entra.microsoft.com` を使用して Entra 管理センターに移動し、**[リソース]** タブの資格情報を使用して **Adele Vance** としてサインインします。

2. 左側のナビゲーションで、**[ID ガバナンス]** を展開し、**[Privileged Identity Management]** を選択します。

3. **[タスク]** で **[自分のロール]** を選択します。

4. **[Microsoft Entra ロール]** タブを選択します。

5. **[対象の割り当て]** で、**[条件付きアクセス管理者]** を検索し、**[アクティブ化]** を選択します。

6. **[アクティブ化]** ペインで、次のように構成します。

    | 設定 | Value |
    |---------|-------|
    | **期間** | 1 時間 |
    | **妥当性** | `Reviewing and updating Conditional Access policies as part of a scheduled security review.` |

7. **[アクティブ化]** を選びます。

    要求が承認待ちであることの確認が表示されます。 このロールはまだアクティブではありません。アクセスを許可するには **sc500-approver** の承認が必要です。

8. このブラウザー ウィンドウは開いたままにしてください。要求が承認された後に戻ってきます。

---

## アクティブ化の要求を承認する

では、**[sc500-approver]** アカウントに切り替え、保留中のアクティブ化の要求を承認します。

1. メインのブラウザー ウィンドウに戻ります (MOD 管理者は現在ログインしています)。

1. Microsoft Entra 管理センターに移動します。

1. 左側のナビゲーションで、**[ID ガバナンス]** を展開し、**[Privileged Identity Management]** を選択します。

1. **[タスク]** で **[要求の承認]** を選択します。

1. **[Microsoft Entra ロール]** のメニュー項目を選択します。

1. **[Adele Vance]** からの **[条件付きアクセス管理者]** ロールの保留中の要求を探します。

1. 要求の横のボックスにマークを追加し、**[承認]** を選択します。

1. **[理由]** フィールドに、`Approved for scheduled security review task.` と入力します。
    ```

1. Select **Submit**.

    You should see an approval message pop-up.

1. You can now minimize this browser window.

---

## Verify the activated role

Return to the **Adele Vance** browser window and verify that the role activation succeeded and grants the expected access.

1. In the **Adele Vance** browser window, refresh the page.

1. In **Privileged Identity Management > My roles > Microsoft Entra roles**, select the **Active assignments** tab.

1. Confirm that **Conditional Access Administrator** appears with a status of **Active** and an expiration time approximately 1 hour from now.

## Test the activation in Conditional Access

1. Look at the menu on the left.

1. In the left navigation, find the **Entra ID** section and select **Conditional Access**.

1. Select **+ Create New policy** to open the policy creation pane.

    > **Note**: If you can open the new policy pane, the role is active and granting the expected permissions. A user without this role would see an error or the option would be unavailable.

1. Select **X** to close the policy pane without saving — creating a policy is not required for this verification step.

---

## Deactivate the role

Just-in-time access means access should be released as soon as the task is complete — not held until the time window expires. You will now manually deactivate the Conditional Access Administrator role for **sc500-user01**.

1. Return to the **Administrator** browser window.

1. Navigate to **Privileged Identity Management > My roles > Microsoft Entra roles > Active assignments**.

1. Find the **Conditional Access Administrator** assignment and select **Deactivate**.

1. In the confirmation dialog, select **Deactivate** again.

1. Confirm the role no longer appears under **Active assignments** and has returned to **Eligible assignments** only.

    The access window is now closed. If sc500-user01 needs to perform CA Admin tasks again, they must submit a new activation request.

---

## Summary

In this lab, you configured Privileged Identity Management to enforce just-in-time access to the Conditional Access Administrator role. You assigned an eligible role, configured activation settings with a time limit, justification requirement, and named approver, then walked through the full activation and approval workflow. You verified that the activated role granted the expected access, and manually deactivated the role to close the access window. You also enabled a system-assigned managed identity on an Azure App Service, establishing the pattern for workload identity that you will apply to Key Vault access in a later lab.

You have successfully completed this exercise.

## Clean up

The lab environment is automatically reset at the end of the session. No manual resource deletion is required.

If you want to clean up the PIM assignment before the session ends:

1. Sign in to the Entra admin center as your Global Administrator.
1. Navigate to **Privileged Identity Management > Microsoft Entra roles > Assignments**.
1. Find the **Conditional Access Administrator** eligible assignment for **sc500-user01**.
1. Select **Remove** and confirm.
