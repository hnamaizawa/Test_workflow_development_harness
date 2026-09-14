---
name: screenplay-to-uia-testcase
friendly_name: ScreenPlay→UI Automation テストケース化
description: "Test Manager のテストケースや手順リストから、まず ScreenPlay（NUITask）で .xaml テストケースを実装し、動作確認後に UI Automation アクティビティ（NTypeInto / NSelectItem / NClick / NGetText）へ差し替え、期待通りに動くか検証する、という3フェーズのワークフローをオーケストレーションする。UiPath Studio Desktop のテスト自動化プロジェクト（ProjectType: Tests / TestAutomationProcess）が対象。『ScreenPlay で作ってから UI Automation に差し替えたい』『Test Manager の手動テストを自動テストにしたい』といった依頼で使用。重い処理は uipath-test（テストケース読取）と uipath-rpa（XAML 作成・UIA キャプチャ・validate/build/run）へ委譲する。"
when_to_use: "ユーザーが ScreenPlay → UI Automation の順でテストを実装・検証したいとき。Test Manager のテストケースまたは手動手順を .xaml 自動テストケースに落とし込みたいとき。ProjectType が Tests / TestAutomationProcess のプロジェクトが対象。単発の XAML 編集や、UI Automation だけ・ScreenPlay だけで良い場合は、このスキルではなく uipath-rpa を直接使う。"
user-invocable: true
---

# ScreenPlay → UI Automation テストケース化

## 概要

UI テストを次の **3フェーズ** で段階的に構築・検証する手順をオーケストレーションするスキル。

1. **ScreenPlay（NUITask）で各手順を実装**し、動作確認する
2. 動作したら **ScreenPlay を UI Automation アクティビティへ差し替える**
3. **期待通りに動作するか検証する**

**なぜ段階的か**：ScreenPlay（AI ベースの自然言語 UI 操作）は素早くテストの筋を通せて画面遷移にも強い。まず ScreenPlay でテスト全体を通してから、より決定論的・高速・堅牢な UI Automation（キャプチャ済みセレクター）へ差し替えることで、確実に動くテストを効率よく作れる。各フェーズはゲート（動作確認）で区切り、フェーズ1が動かないうちはフェーズ2へ進まない。

**委譲方針**：このスキルは順序・ゲート・落とし穴の管理に徹し、重い処理は既存スキルへ委譲する（処理を重複させない）。

| 委譲先スキル | 担当 |
|---|---|
| **uipath-test** | Test Manager のテストケース・手順（steps）の読取（`uip tm`） |
| **uipath-rpa** | .xaml テストケースの作成・編集、UI Automation のターゲットキャプチャ、validate / build / run |

各フェーズで該当スキルの `SKILL.md` を読んでから実行する（**一度に1スキル**、先読みしない）。

## 前提条件

- 対象は XAML テスト自動化プロジェクト（`project.json` の `projectType` が Tests、Studio 上の ProjectType: TestAutomationProcess）。
- ScreenPlay（NUITask）と UI Automation（NTypeInto 等）はいずれも assembly `UiPath.UIAutomationNext.Activities`、名前空間 `xmlns:uix="http://schemas.uipath.com/workflow/activities/uix"`。ルート要素にこの名前空間宣言が必要。
- フェーズ2以降は `UiPath.UIAutomation.Activities` パッケージが必要（uipath-rpa の § UIA Prerequisites / Rule 7・7a を参照。バージョン下限とインストール同意の扱いはそこが唯一の正）。
- Test Manager から手順を取得する場合は `uip login` 済みであること（uipath-test のルール1・2）。
- Studio Desktop 内で動作する。**Studio を kill/restart するコマンドは実行しない**（セッションが切れる）。

## 入力パラメータ

- **テストケースの出所**：Test Manager のテストケース（`project-key` ＋ `test-case-id`/`key`）、または手動手順リスト（テキスト）。TM の場合は uipath-test で手順（steps）を取得する。
- **対象アプリ / URL**：テスト対象の Web／デスクトップアプリ。フェーズ2のキャプチャで使用。
- **出力先の .xaml**：対象プロジェクトのテストケースファイル（既存 or 新規）。新規なら `project.json` の `designOptions.fileInfoCollection` へ登録（uipath-rpa Common Rule 10）。
- **パラメータ化する値**：テストデータ（メールアドレス、金額など）は **引数化**し、既定値を持たせる（データ駆動テストへ流用可能にする）。

## ワークフロー（3フェーズ）

**進め方の原則**：各フェーズの末尾に「動作確認ゲート」を置く。ゲートを通らない限り次フェーズへ進まない。TodoTool で3フェーズ＋各ゲートを管理する。

### フェーズ1：ScreenPlay（NUITask）で各手順を実装

1. **手順の取得**。Test Manager が出所なら uipath-test の SKILL.md を読み、`uip tm testcases steps list --project-key <KEY> --test-case-id <UUID> --output json` で手順を取得。手動手順ならユーザー提示のリストを使う。
2. **設計**。1手順 = 1 NUITask、並び順は手順順。全 NUITask を **単一の NApplicationCard** 配下に置く（ScreenPlay は AI が画面遷移をまたいで適応するため 1 カードで良い）。Given-When-Then 構成にする（uipath-rpa の testing-guide）。
3. **NUITask の書き方**（詳細は `references/phase1-screenplay.md`）：
   - `DisplayName` = 手順名（日本語のまま）
   - `Prompt` = その手順を平易な日本語 1 文で指示（例：「Loan Amount Requested に 60000 を入力する」）
   - 検証系の手順は「〜であることを確認する」と表現
   - テストデータは引数化（`&引数&` の形で参照 → Studio が placeholder に正規化）
4. **XAML 作成**。uipath-rpa の SKILL.md を読み、Rule 7（UIA 前提）に従う。NUITask は class `UiPath.Semantic.Activities.NUITask`、ルートに `xmlns:uix` 宣言。新規テストケースなら `project.json` へ登録（Common Rule 10）。
5. **ゲート：動作確認**。per-file validate → project build（uipath-rpa Rule 3）。必要なら `uip rpa run` で実機確認。**ここが通って初めてフェーズ2へ**。

### フェーズ2：UI Automation アクティビティへ差し替え

フェーズ1が動作した場合のみ実施。

1. **マッピング**（詳細は `references/phase2-uia-conversion.md`）：各 NUITask を対応する UIA アクティビティへ置換。
   - 入力 → `NTypeInto`
   - ドロップダウン選択 → `NSelectItem`
   - クリック → `NClick`
   - テキスト取得／検証 → `NGetText`（出力は `<uix:NGetText.Text>` に `OutArgument<String>` で束縛。`Value` は不可。メンバ名はパッケージ版により `Text`／`TextString` の別あり — 現物確認）
2. **セレクターのキャプチャ**。**手書き禁止**。uipath-rpa Rule 7 に従い `uia-configure-target` でキャプチャする。既にキャプチャ済みの実装（例：過去ビルドの `.local/Compiler` や Object Repository）があれば、そのセレクターを再利用してよい。
3. **カード構造**。画面（ウィンドウスコープ）ごとに `NApplicationCard`（`Version="V2"`）を分ける。ScreenPlay の 1 カードが、UIA では画面数ぶんの複数カードになり得る（申請画面／結果画面 など）。
4. **引数バインドの維持**。フェーズ1で引数化した値は UIA 版でも引数参照を保つ（例：`Text="[メールアドレス]"`）。
5. **ゲート：動作確認**。per-file validate → project build。

### フェーズ3：期待通りに動作するか検証

1. `uip rpa run`（または `debug start`）で実行（uipath-rpa の uia-starter-guide の run/debug 手順）。成否は外側の `Result` / `HasErrors` で判定する（ログの `Level` では判定しない — Rule 8a）。
2. セレクター不一致など実機差異があれば `uia-configure-target` で再キャプチャして調整。
3. 詳細・落とし穴は `references/phase3-verify.md`。

## 重要ルール / ガードレール

- **フェーズは飛ばさない**。フェーズ1が validate/build（＋必要なら run）を通るまでフェーズ2へ進まない。
- **セレクターは絶対に手書きしない**（uipath-rpa Rule 7）。キャプチャ、または既存キャプチャ済み資産の再利用のみ。
- **日本語文字列を PowerShell のコマンド本文に直書きしない**（エンコーディング破損の実績あり）。XAML 等の日本語コンテンツは WriteFile（UTF-8）で書く。CLI にパスを渡すときは対象フォルダへ `Set-Location -LiteralPath` してから ASCII 相対パスで実行すると安定する。
- **Studio を kill/restart しない**（HostConstraint）。
- 破壊的操作（削除・全消去）を提案・実行しない。
- ScreenPlay 版と UIA 版で差し替えても名前空間・アセンブリ参照（`UiPath.UIAutomationNext.Activities`）は共通。
- 変換は「置き換え」であり、テストの意図（手順・検証内容）を変えない。

## 参照ファイル

- `references/phase1-screenplay.md` — NUITask 作成規則と注釈付き XAML テンプレート、シリアライズ形式
- `references/phase2-uia-conversion.md` — NUITask→UIA マッピング、キャプチャ手順、複数カード構造、UIA XAML テンプレート
- `references/phase3-verify.md` — validate→build→run の検証ループ、エンコーディング注意、引数バインド
- `references/xaml-examples.md` — ScreenPlay 完全例 ＋ UI Automation 完全例（実 TestCase.xaml 準拠）
