# Test_workflow_development_harness

このプロジェクトは、**UI テストを ScreenPlay から UIA（UI Automation）へ段階的に移行する「開発工場」** です。

---

## 📋 プロジェクトの概要

| 項目 | 詳細 |
|------|------|
| **種類** | TestAutomationProcess（UiPath Test Manager 連携） |
| **フレームワーク** | Windows UI Automation（VisualBasic） |
| **主テストケース** | `TestCase.xaml` |
| **テスト対象** | UiBank ローン申請フロー（Edge ブラウザ） |
| **引数** | メールアドレス（既定値: `demo@demo.com`） |
| **結果** | `resultText` 変数に取得 |

---

## 🔄 3フェーズの流れ

```
Phase 1: ScreenPlay で実装
    ↓
Phase 2: 動作確認
    ↓
Phase 3: UI Automation へ差し替え + 再度動作確認
    ↓
完成（再利用可能なテストケース）
```

---

## 🛠 テストケースの構成

**TestCase.xaml** には以下が含まれています：

| 項目 | 内容 |
|------|------|
| **UI アクティビティ** | `NTypeInto` ×4、`NSelectItem` ×1、`NClick` ×1、`NGetText` ×1 |
| **入力フィールド** | Loan Amount、Email（メール欄）、Current Yearly Income、Age |
| **セレクター** | キャプチャ済み（手書き不可） |
| **画面スコープ** | 2枚の `NApplicationCard`（申請画面 + 結果画面） |

---

## ⚙️ 検証・実行コマンド

```powershell
# 文字化け対策
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
Set-Location -LiteralPath "<プロジェクトのパス>"

# 1. 単一ファイルの検証
uip rpa validate --file-path "TestCase.xaml" --project-dir "." --output json

# 2. プロジェクト全体のビルド
uip rpa build "." --output json

# 3. テスト実行
uip rpa run "TestCase.xaml" --project-dir "." --output json
```

---

## ⚠️ 重要な注意事項（落とし穴）

| 注意 | 理由 |
|------|------|
| **日本語を PowerShell に直書きしない** | 文字化けする → `WriteFile`（UTF-8）で別ファイルに書いて挿入 |
| **セレクターを手書きしない** | キャプチャツール `uia-configure-target` を使う |
| **Studio を kill/restart しない** | セッションが切れて UI 要素が失われる |
| **WriteFile の配置場所** | プロジェクトフォルダ内のみ → 外側は `Copy-Item` で配置 |

---

## 📚 関連スキル

専用スキル **`screenplay-to-uia-testcase`** で自動化可能：
- ソース: `skills/screenplay-to-uia-testcase/SKILL.md`
- オーケストレーション、NUITask の書き方、検証ループを自動処理

詳細は [AGENTS.md](./AGENTS.md) を参照してください。
