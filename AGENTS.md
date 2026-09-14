# AGENTS.md — Test_Cloudプロジェクト（ScreenPlay→UIA 安定開発工場）

このプロジェクトは、Test Manager のテストケースから **ScreenPlay で実装 → 動作確認 → UI Automation へ差し替え → 動作確認** という3フェーズで UI テストを作る「開発工場」です。この手順は再利用可能なスキルとして体系化されています。

## この手順を再現するには

専用スキル **`screenplay-to-uia-testcase`** を使う。
- ソース: `skills/screenplay-to-uia-testcase/SKILL.md`（このプロジェクト内）
- ハーネス配置: `.autopilot/skills/screenplay-to-uia-testcase/`（自動認識用のコピー）
- 内容: 3フェーズのオーケストレーション、NUITask／UIA の書き方、検証ループ、落とし穴。重い処理は uipath-test（テストケース読取）と uipath-rpa（XAML 作成・UIA キャプチャ・validate/build/run）へ委譲。

<!-- PROJECT-CONTEXT:START -->
## プロジェクト概要

- **ProjectType**: TestAutomationProcess（Tests）／ **Framework**: Windows ／ **式言語**: VisualBasic
- **主要テストケース**: `TestCase.xaml`（`project.json` の `designOptions.fileInfoCollection` 登録済み）
- **引数**: `メールアドレス`（String、既定 `demo@demo.com`）／ **変数**: `resultText`（String）
- **依存パッケージ**: UiPath.System.Activities / UiPath.Testing.Activities / UiPath.UIAutomation.Activities

## 由来（今回の実装）

- Test Manager プロジェクト **HNTS**（Namaizawa_Test_Project）のテストケース **HNTS:5**「ローン申請_全項目入力完了と正常送信」（7手順）から作成。
- 対象アプリ: UiBank ローン申請（申請 `https://uibank.uipath.com/loans/apply` → 結果 `.../loans/result/...`、BrowserType: Edge）。

## TestCase.xaml の現状（UI Automation 版）

- 構成: **2枚の NApplicationCard**（申請画面／結果画面）。ScreenPlay 版は1枚だったが、UIA はウィンドウスコープが画面ごとに異なるため分割している。
- アクティビティ: `NTypeInto` ×4（Loan Amount / Email / Current Yearly Income / Age）、`NSelectItem` ×1（Loan Term）、`NClick` ×1（Submit）、`NGetText` ×1（結果 → `resultText`）。
- メール欄は引数バインド: `Text="[メールアドレス]"`。
- 検証済み: per-file validate = `No diagnostics found`、project build = `Success`。

## 検証コマンド（日本語パス対策込み）

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
Set-Location -LiteralPath "<このプロジェクトのパス>"
uip rpa validate --file-path "TestCase.xaml" --project-dir "." --output json
uip rpa build "." --output json
uip rpa run  "TestCase.xaml" --project-dir "." --output json
```

## 注意（落とし穴）

- 日本語文字列を PowerShell コマンド本文に直書きしない（文字化け）。日本語コンテンツは WriteFile（UTF-8）で書く。
- WriteFile はプロジェクトフォルダ内限定。プロジェクト外（`.autopilot/skills` 等）へは WriteFile → PowerShell `Copy-Item`（バイト保持）で配置する。
- セレクターは手書きしない。`uia-configure-target` でキャプチャ、または既存キャプチャ済み資産を流用。
- Studio を kill/restart しない（セッションが切れる）。
<!-- PROJECT-CONTEXT:END -->
