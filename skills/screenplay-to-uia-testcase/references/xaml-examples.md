# 実行サンプル（完全な XAML 例）

題材は HNTS:5「ローン申請_全項目入力完了と正常送信」（UiBank ローン申請）。フェーズ1（ScreenPlay）とフェーズ2（UI Automation）の完全なテストケース XAML を示す。

**読み方**
- 両フェーズは**同じヘッダー**（root 要素・名前空間・`x:Members`・NamespacesForImplementation/ReferencesForImplementation）を共有する。body（root `Sequence` 以下）だけが異なる。
- UI Automation のターゲット（セレクター）は `uia-configure-target` の**キャプチャ産物**で手書きしない。UIA 例では実キャプチャのターゲットを掲載し、長いものは読みやすさのため属性を一部簡略化している。
- **検証済みの完全ファイル**はこのプロジェクトの `TestCase.xaml`（UIA 版）を直接参照するのが最も確実。

## 共通ヘッダー（両フェーズ共通）

```xml
<Activity mc:Ignorable="sap sap2010" x:Class="TestCase" this:TestCase.メールアドレス="demo@demo.com"
    xmlns="http://schemas.microsoft.com/netfx/2009/xaml/activities"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:sap="http://schemas.microsoft.com/netfx/2009/xaml/activities/presentation"
    xmlns:sap2010="http://schemas.microsoft.com/netfx/2010/xaml/activities/presentation"
    xmlns:scg="clr-namespace:System.Collections.Generic;assembly=System.Private.CoreLib"
    xmlns:sco="clr-namespace:System.Collections.ObjectModel;assembly=System.Private.CoreLib"
    xmlns:this="clr-namespace:"
    xmlns:ui="http://schemas.uipath.com/workflow/activities"
    xmlns:uix="http://schemas.uipath.com/workflow/activities/uix"
    xmlns:uta="clr-namespace:UiPath.Testing.Activities;assembly=UiPath.Testing.Activities"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <x:Members>
    <x:Property Name="メールアドレス" Type="InArgument(x:String)" />
  </x:Members>
  <!-- ここに Studio 自動生成の NamespacesForImplementation / ReferencesForImplementation が入る
       （UiPath.UIAutomationNext.Activities / UiPath.Testing.Activities などを含む。TestCase.xaml と同一） -->
  <!-- ↓ body（root Sequence 以下）はフェーズごとに異なる。次節参照 -->
</Activity>
```

- `x:Class="TestCase"` と `this:TestCase.メールアドレス="demo@demo.com"`（引数の既定値）、`xmlns:this="clr-namespace:"` はセットで必要。
- ルートに `xmlns:uix`（UIA/ScreenPlay 共通）が必須。検証を使うなら `xmlns:uta` も。

## フェーズ1：ScreenPlay（NUITask）完全例（body）

単一の NApplicationCard 配下に、1手順=1 NUITask を並べる（ScreenPlay は AI が画面遷移を吸収するため単一カードで良い）。

```xml
<Sequence DisplayName="ローン申請_全項目入力完了と正常送信">
  <Sequence.Variables>
    <Variable x:TypeArguments="x:String" Name="resultText" />
  </Sequence.Variables>

  <!-- Given -->
  <Sequence DisplayName="前提条件 Given">
    <ui:Comment Text="前提条件: Edge で UiBank ローン申請画面 (https://uibank.uipath.com/loans/apply) が表示可能" />
  </Sequence>

  <!-- When + Then -->
  <uix:NApplicationCard DisplayName="アプリを使用: UiBank ローン申請" Version="V2">
    <uix:NApplicationCard.TargetApp>
      <uix:TargetApp Url="https://uibank.uipath.com/loans/apply" BrowserType="Edge" />
    </uix:NApplicationCard.TargetApp>
    <uix:NApplicationCard.Body>
      <ActivityAction x:TypeArguments="x:Object">
        <ActivityAction.Argument>
          <DelegateInArgument x:TypeArguments="x:Object" Name="WSSessionData" />
        </ActivityAction.Argument>
        <Sequence DisplayName="実行">

          <uix:NUITask DisplayName="Loan Amount Requested に入力">
            <uix:NUITask.Prompt>
              <scg:List x:TypeArguments="InArgument(x:String)" Capacity="1">
                <InArgument x:TypeArguments="x:String">["Loan Amount Requested に 60000 を入力する"]</InArgument>
              </scg:List>
            </uix:NUITask.Prompt>
          </uix:NUITask>

          <uix:NUITask DisplayName="Email Address of Requester に入力">
            <uix:NUITask.Prompt>
              <scg:List x:TypeArguments="InArgument(x:String)" Capacity="2">
                <InArgument x:TypeArguments="x:String">["Email Address of Requester に {0} を入力する"]</InArgument>
                <InArgument x:TypeArguments="x:String">[メールアドレス]</InArgument>
              </scg:List>
            </uix:NUITask.Prompt>
          </uix:NUITask>

          <uix:NUITask DisplayName="Loan Term を選択">
            <uix:NUITask.Prompt>
              <scg:List x:TypeArguments="InArgument(x:String)" Capacity="1">
                <InArgument x:TypeArguments="x:String">["Loan Term のドロップダウンから 5 を選択する"]</InArgument>
              </scg:List>
            </uix:NUITask.Prompt>
          </uix:NUITask>

          <uix:NUITask DisplayName="Current Yearly Income に入力">
            <uix:NUITask.Prompt>
              <scg:List x:TypeArguments="InArgument(x:String)" Capacity="1">
                <InArgument x:TypeArguments="x:String">["Current Yearly Income (Before Taxes) に 50000 を入力する"]</InArgument>
              </scg:List>
            </uix:NUITask.Prompt>
          </uix:NUITask>

          <uix:NUITask DisplayName="Age に入力">
            <uix:NUITask.Prompt>
              <scg:List x:TypeArguments="InArgument(x:String)" Capacity="1">
                <InArgument x:TypeArguments="x:String">["Age に 35 を入力する"]</InArgument>
              </scg:List>
            </uix:NUITask.Prompt>
          </uix:NUITask>

          <uix:NUITask DisplayName="Submit Loan Application をクリック">
            <uix:NUITask.Prompt>
              <scg:List x:TypeArguments="InArgument(x:String)" Capacity="1">
                <InArgument x:TypeArguments="x:String">["Submit Loan Application ボタンをクリックする"]</InArgument>
              </scg:List>
            </uix:NUITask.Prompt>
          </uix:NUITask>

          <uix:NUITask DisplayName="受付結果を確認">
            <uix:NUITask.Prompt>
              <scg:List x:TypeArguments="InArgument(x:String)" Capacity="1">
                <InArgument x:TypeArguments="x:String">["受付番号と受付ステータスが表示されていることを確認する"]</InArgument>
              </scg:List>
            </uix:NUITask.Prompt>
          </uix:NUITask>

        </Sequence>
      </ActivityAction>
    </uix:NApplicationCard.Body>
  </uix:NApplicationCard>
</Sequence>
```

## フェーズ2：UI Automation 完全例（body）

画面（ウィンドウスコープ）ごとに `NApplicationCard` を分ける。ここでは申請画面カードと結果画面カードの2枚。各アクティビティは実 `TestCase.xaml`（ビルド成功済み）の形に準拠。

```xml
<Sequence DisplayName="ローン申請_全項目入力完了と正常送信">
  <Sequence.Variables>
    <Variable x:TypeArguments="x:String" Name="resultText" />
  </Sequence.Variables>

  <!-- Given -->
  <Sequence DisplayName="前提条件 Given">
    <ui:Comment Text="前提条件: Edge で UiBank ローン申請画面が表示可能" />
  </Sequence>

  <!-- When: 申請画面カード -->
  <Sequence DisplayName="操作 When">
    <uix:NApplicationCard DisplayName="Edge: UiBank-Loan Apply" HealingAgentBehavior="Job"
        ScopeGuid="65e567eb-a1f7-4205-87ab-09559e140c54" Version="V2">
      <uix:NApplicationCard.Body>
        <ActivityAction x:TypeArguments="x:Object">
          <ActivityAction.Argument>
            <DelegateInArgument x:TypeArguments="x:Object" Name="WSSessionData" />
          </ActivityAction.Argument>
          <Sequence DisplayName="実行">

            <!-- 入力（メールは引数バインド）。ターゲットは実キャプチャ（属性を一部簡略化） -->
            <uix:NTypeInto ActivateBefore="True" ClickBeforeMode="Single" DisplayName="文字を入力 'Email Address of Requester'"
                EmptyFieldMode="SingleLine" HealingAgentBehavior="SameAsCard"
                ScopeIdentifier="65e567eb-a1f7-4205-87ab-09559e140c54" Text="[メールアドレス]" Version="V5">
              <uix:NTypeInto.Target>
                <uix:TargetAnchorable ContentHash="7iki3_Zi4U2TQYKeciBiog" ElementType="InputBox"
                    ElementVisibilityArgument="Interactive"
                    FuzzySelectorArgument="&lt;webctrl id='email' tag='INPUT' type='email' class='form-control uibank-input' matching:id='fuzzy' fuzzylevel:id='0.0' /&gt;"
                    Guid="9479a71e-fd1b-4a36-b141-3bdb53d63e50"
                    ScopeSelectorArgument="&lt;html app='msedge.exe' title='UiBank-Loan Apply' /&gt;"
                    SearchSteps="FuzzySelector" Version="V6" WaitForReadyArgument="Interactive">
                  <uix:TargetAnchorable.Anchors>
                    <scg:List x:TypeArguments="uix:ITarget" Capacity="1">
                      <uix:Target ElementType="Text"
                          FuzzySelectorArgument="&lt;webctrl tag='LABEL' class='uibank-label' aaname='Email Address of Requester' check:text='Email Address of Requester' /&gt;"
                          Guid="eced1f4b-eb65-4914-8508-d4c5de8ea2a6" SearchSteps="FuzzySelector" />
                    </scg:List>
                  </uix:TargetAnchorable.Anchors>
                </uix:TargetAnchorable>
              </uix:NTypeInto.Target>
            </uix:NTypeInto>

            <!-- 以降のターゲットは同じ形（uia-configure-target で生成）。ここでは省略 -->
            <uix:NTypeInto DisplayName="文字を入力 'Loan Amount Requested'" HealingAgentBehavior="SameAsCard"
                ScopeIdentifier="65e567eb-a1f7-4205-87ab-09559e140c54" Text="60000" Version="V5">
              <uix:NTypeInto.Target><!-- captured target --></uix:NTypeInto.Target>
            </uix:NTypeInto>

            <uix:NSelectItem DisplayName="項目を選択 'Loan Term'" HealingAgentBehavior="SameAsCard"
                Item="5" ScopeIdentifier="65e567eb-a1f7-4205-87ab-09559e140c54" Version="V5">
              <uix:NSelectItem.Target><!-- captured target --></uix:NSelectItem.Target>
            </uix:NSelectItem>

            <uix:NTypeInto DisplayName="文字を入力 'Current Yearly Income'" HealingAgentBehavior="SameAsCard"
                ScopeIdentifier="65e567eb-a1f7-4205-87ab-09559e140c54" Text="50000" Version="V5">
              <uix:NTypeInto.Target><!-- captured target --></uix:NTypeInto.Target>
            </uix:NTypeInto>

            <uix:NTypeInto DisplayName="文字を入力 'Age'" HealingAgentBehavior="SameAsCard"
                ScopeIdentifier="65e567eb-a1f7-4205-87ab-09559e140c54" Text="35" Version="V5">
              <uix:NTypeInto.Target><!-- captured target --></uix:NTypeInto.Target>
            </uix:NTypeInto>

            <uix:NClick ActivateBefore="True" ClickType="Single" DisplayName="クリック 'Submit Loan Application'"
                HealingAgentBehavior="SameAsCard" KeyModifiers="None" MouseButton="Left"
                ScopeIdentifier="65e567eb-a1f7-4205-87ab-09559e140c54" Version="V5">
              <uix:NClick.Target><!-- captured target --></uix:NClick.Target>
            </uix:NClick>

          </Sequence>
        </ActivityAction>
      </uix:NApplicationCard.Body>
    </uix:NApplicationCard>
  </Sequence>

  <!-- Then: 結果画面カード -->
  <Sequence DisplayName="検証 Then">
    <uix:NApplicationCard DisplayName="UiBank-Loan result" HealingAgentBehavior="Job"
        ScopeGuid="a1b2c3d4-e5f6-7890-abcd-ef1234567890" Version="V2">
      <uix:NApplicationCard.Body>
        <ActivityAction x:TypeArguments="x:Object">
          <ActivityAction.Argument>
            <DelegateInArgument x:TypeArguments="x:Object" Name="WSSessionData" />
          </ActivityAction.Argument>
          <Sequence DisplayName="実行">
            <!-- NGetText: 出力は .Text 要素に OutArgument で束縛（TextString 属性ではない） -->
            <uix:NGetText DisplayName="テキストを取得 '受付結果'" HealingAgentBehavior="SameAsCard"
                ScopeIdentifier="a1b2c3d4-e5f6-7890-abcd-ef1234567890" Version="V5">
              <uix:NGetText.Target>
                <uix:TargetAnchorable ContentHash="DNQYIL5dv0CvL7jUQaP-NA" ElementType="Text"
                    ElementVisibilityArgument="Interactive"
                    FullSelectorArgument="&lt;webctrl idx='2' tag='H1' /&gt;"
                    Guid="9af3609f-2f95-4c82-9a3b-ced3671c56b7"
                    ScopeSelectorArgument="&lt;html app='msedge.exe' title='UiBank-Loan result' /&gt;"
                    SearchSteps="Selector" Version="V6" WaitForReadyArgument="Interactive" />
              </uix:NGetText.Target>
              <uix:NGetText.Text>
                <OutArgument x:TypeArguments="x:String">[resultText]</OutArgument>
              </uix:NGetText.Text>
            </uix:NGetText>

            <ui:LogMessage DisplayName="受付結果テキストを出力" Level="Info" Message="[resultText]" />

            <uta:VerifyExpression ContinueOnFailure="True"
                DisplayName="検証: 受付画面に Approved が含まれること（大文字小文字無視）"
                Expression="[resultText.ToLower().Contains(&quot;approved&quot;)]"
                TakeScreenshotInCaseOfFailingAssertion="True"
                TakeScreenshotInCaseOfSucceedingAssertion="False" />
          </Sequence>
        </ActivityAction>
      </uix:NApplicationCard.Body>
    </uix:NApplicationCard>
  </Sequence>
</Sequence>
```

**ポイント**
- メール欄は `Text="[メールアドレス]"`（引数バインド）。他は `Text="60000"` のようにリテラル。
- **`NGetText` の出力は `<uix:NGetText.Text>` に `OutArgument<String>` で束縛**（例：`[resultText]`）。`Value` は存在しない。属性形 `TextString` はパッケージ版により異なるため、必ず `activities get-default-xaml` ＋ `<Activity>.md` で現物確認する。
- 各ターゲットは `uia-configure-target` の産物。**手書きしない**。掲載した Email／NGetText のターゲット構造（`TargetAnchorable` ＋ `FuzzySelectorArgument`／`FullSelectorArgument`／`ScopeSelectorArgument`／`Anchors`）が実際の形。
- 検証は `UiPath.Testing.Activities` の `VerifyExpression`（名前空間 `uta`）。ルートに `xmlns:uta` 宣言が必要。
- 上記は骨格。ViewState（`HintSize`／`IdRef`）等は Studio が付与する。**検証済みの完全形はこのプロジェクトの `TestCase.xaml`**。
