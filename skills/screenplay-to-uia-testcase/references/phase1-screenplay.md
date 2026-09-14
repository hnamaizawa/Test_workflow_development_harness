# フェーズ1詳細：ScreenPlay（NUITask）で各手順を実装

## NUITask とは

- ScreenPlay アクティビティの実体は class **`UiPath.Semantic.Activities.NUITask`**、assembly `UiPath.UIAutomationNext.Activities`。
- 自然言語のプロンプト（Prompt）を AI が解釈して UI 操作を実行する。**セレクター不要**で、画面遷移をまたいでも適応する。
- そのため、まず ScreenPlay で「テストの筋」を素早く通すのに向く。

## 作成規則

1. **1手順 = 1 NUITask**。テストケースの各ステップを1つの NUITask に対応させる。
2. **DisplayName = 手順名**（日本語のまま）。Test Manager の手順名をそのまま使う。
3. **Prompt = 平易な日本語 1 文**。「何を・どうする」を明確に。
   - 入力：「Loan Amount Requested に 60000 を入力する」
   - 選択：「Loan Term のドロップダウンから 5 を選択する」
   - クリック：「Submit Loan Application ボタンをクリックする」
   - 検証：「受付番号と受付ステータスが表示されていることを確認する」
4. **検証系の手順は「〜であることを確認する」**と表現する（アサーションの意図を明示）。
5. **テストデータは引数化**。ハードコードせず引数参照にする。Prompt 内では `&引数名&` の形で書くと、Studio が placeholder に正規化する。

## 配置構造（Given-When-Then）

全 NUITask を **単一の NApplicationCard**（Use Application/Browser）配下の Sequence に置く。ScreenPlay は AI が画面遷移をまたいで適応するため 1 カードで良い。

- **Given**：対象アプリ/ブラウザーを開く（NApplicationCard の TargetApp）
- **When**：操作系の NUITask（入力・選択・クリック）
- **Then**：検証系の NUITask ＋ 必要なら VerifyExpression / LogMessage

（uipath-rpa の testing-guide § XAML Test Case Structure を参照）

## シリアライズ形式（重要）

Studio は作成時の `Task=` 属性を **`Prompt`** に正規化する。Prompt は `List<InArgument<String>>` としてシリアライズされる。パラメータ化した箇所は `{0}`（string.Format 形式）のプレースホルダとして本文に入り、対応する引数がリストに続く。

```xml
<uix:NUITask DisplayName="Email Address of Requester に入力">
  <uix:NUITask.Prompt>
    <scg:List x:TypeArguments="InArgument(x:String)" Capacity="2">
      <InArgument x:TypeArguments="x:String">["Email Address of Requester に {0} を入力する"]</InArgument>
      <InArgument x:TypeArguments="x:String">[メールアドレス]</InArgument>
    </scg:List>
  </uix:NUITask.Prompt>
</uix:NUITask>
```

パラメータの無い手順は要素1つだけ：

```xml
<uix:NUITask DisplayName="Loan Amount Requested に入力">
  <uix:NUITask.Prompt>
    <scg:List x:TypeArguments="InArgument(x:String)" Capacity="1">
      <InArgument x:TypeArguments="x:String">["Loan Amount Requested に 60000 を入力する"]</InArgument>
    </scg:List>
  </uix:NUITask.Prompt>
</uix:NUITask>
```

ルート要素に次の名前空間宣言が必要：
- `xmlns:uix="http://schemas.uipath.com/workflow/activities/uix"`
- `xmlns:scg="clr-namespace:System.Collections.Generic;assembly=System.Private.CoreLib"`（環境の既定に合わせる）

## NApplicationCard（V2）の骨組み

```xml
<uix:NApplicationCard DisplayName="アプリを使用" Version="V2">
  <uix:NApplicationCard.TargetApp>
    <uix:TargetApp Url="https://.../apply" BrowserType="Edge" />
  </uix:NApplicationCard.TargetApp>
  <uix:NApplicationCard.Body>
    <ActivityAction x:TypeArguments="x:Object">
      <ActivityAction.Argument>
        <DelegateInArgument x:TypeArguments="x:Object" Name="WSSessionData" />
      </ActivityAction.Argument>
      <Sequence>
        <!-- ここに NUITask を手順順に並べる -->
      </Sequence>
    </ActivityAction>
  </uix:NApplicationCard.Body>
</uix:NApplicationCard>
```

- `Version="V2"` は必須。Body は `ActivityAction x:TypeArguments="x:Object"` ＋ `DelegateInArgument Name="WSSessionData"` → Sequence の形。
- コンテナ本体は単一アクティビティでも `<Sequence>` で包む（uipath-rpa Rule 24）。

## 引数の定義

引数化した値（例：メールアドレス）は `x:Members` にプロパティとして定義し、既定値を持たせる：

```xml
<x:Members>
  <x:Property Name="メールアドレス" Type="InArgument(x:String)" />
</x:Members>
```
既定値はルートタグ属性で与える（例：`this:TestCase.メールアドレス="demo@demo.com"`、`xmlns:this="clr-namespace:"` が必要）。`InArgument<string>` の裸の属性値はリテラル扱い。

## 手順取得（Test Manager が出所の場合）

uipath-test の SKILL.md を読んでから：
```
uip tm testcases steps list --project-key <KEY> --test-case-id <UUID> --output json
```
- `--test-case-id` は **UUID**（`uip tm testcases list` の `Id`）。`--test-case-key`（`PROJECT:NUM`）とは別物なので混同しない。

## 完了ゲート

uipath-rpa Rule 3：per-file `validate` → project `build`。必要なら `uip rpa run` で実機確認。詳細は `phase3-verify.md`。**ここを通って初めてフェーズ2へ進む。**
