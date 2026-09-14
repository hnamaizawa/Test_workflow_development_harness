# フェーズ2詳細：UI Automation アクティビティへ差し替え

**前提**：フェーズ1（ScreenPlay 版）が per-file validate → project build（＋必要なら run）を通過していること。動いていないうちは差し替えない。

## なぜ差し替えるか

ScreenPlay（AI 実行）は柔軟だが実行時に AI 推論が入る。UI Automation は**キャプチャ済みセレクター**で決定論的・高速・堅牢。テストの筋が通ったら UIA に置き換えて安定運用する。

## NUITask → UIA アクティビティ 対応表

| 手順の種類 | ScreenPlay | UI Automation | 主要プロパティ |
|---|---|---|---|
| テキスト入力 | NUITask | `NTypeInto` | `Text`（入力値。引数参照可） |
| ドロップダウン選択 | NUITask | `NSelectItem` | `Item`（選択値） |
| クリック | NUITask | `NClick` | （ターゲットのみ） |
| テキスト取得・検証 | NUITask | `NGetText` | 出力は `<uix:NGetText.Text>` に `OutArgument<String>`（`Value` 不可） |

いずれも assembly `UiPath.UIAutomationNext.Activities`、名前空間 `uix`。

## セレクターは手書きしない（最重要）

- uipath-rpa **Rule 7** に従い、着手前に**全文を読む**：uipath-rpa の `uia-starter-guide.md` ＋ UIA パッケージガイド `{PROJECT_DIR}/.local/docs/packages/UiPath.UIAutomation.Activities/ui-automation-guide.md`。
- ターゲットは **`uia-configure-target`** でキャプチャする。`FuzzySelectorArgument` / `FullSelectorArgument` / `ScopeSelectorArgument` を手打ちしない（幻覚セレクターの主因）。
- **再利用の近道**：同じ画面の UIA 実装が既にプロジェクトにあるなら（過去ビルドの `.local/Compiler` 配下や Object Repository の `.objects`）、そのキャプチャ済みセレクターを**そのまま流用**してよい。400〜600 文字級のセレクター文字列を手で打ち直さない（変換スクリプトでコピーする等、機械的に移す）。

## カード構造：画面ごとに NApplicationCard を分ける

- ScreenPlay は **1 カード**で足りた（AI が画面遷移を吸収）。
- UIA は**ウィンドウスコープ（画面）ごとにセレクターが異なる**ため、画面数ぶんの `NApplicationCard`（`Version="V2"`）に分割する。
  - 例：申請画面カード（入力・選択・クリック）＋ 結果画面カード（結果テキスト取得）。
- 各カードの ScopeGuid / TargetApp（`Url`・`BrowserType`）は該当画面のものにする。

## アクティビティ雛形（キャプチャ済みターゲット前提）

`activities get-default-xaml` は型既定値のプロパティを省く。特に**出力プロパティが危険**（Rule 21 の skip-tax）。必ず各 `.md`（`{PROJECT_DIR}/.local/docs/packages/UiPath.UIAutomation.Activities/activities/<Activity>.md`）を読んでプロパティを補う。

**NTypeInto**（裸の属性値はリテラル。引数参照は `[引数名]`）：
```xml
<uix:NTypeInto DisplayName="Loan Amount Requested に入力" Text="60000">
  <uix:NTypeInto.Target>
    <!-- uia-configure-target でキャプチャ済みのターゲット -->
  </uix:NTypeInto.Target>
</uix:NTypeInto>
```
メール欄は引数参照：`Text="[メールアドレス]"`（`Text=` プレフィックスで引数の既定値と区別）。

**NSelectItem**（ドロップダウン）：`Item="5"` ＋ Target。
**NClick**（クリック）：Target のみ。
**NGetText**（結果取得。出力は `<uix:NGetText.Text>` 要素に `OutArgument<String>`）：
```xml
<uix:NGetText DisplayName="受付結果を取得">
  <uix:NGetText.Target>
    <!-- キャプチャ済みターゲット -->
  </uix:NGetText.Target>
  <uix:NGetText.Text>
    <OutArgument x:TypeArguments="x:String">[resultText]</OutArgument>
  </uix:NGetText.Text>
</uix:NGetText>
```

## 引数バインドを維持

フェーズ1で引数化した値は UIA 版でも引数参照を保つ。`x:Members` のプロパティ定義とルート属性の既定値もそのまま維持する（例：`メールアドレス` 既定 `demo@demo.com`）。**差し替えでテストの意図（手順・検証内容）を変えない。**

## 差し替えの実務（大きな XAML を安全に）

- 巨大な XAML を1回の書き込みで生成すると途中で切れる恐れがある。セクション単位で分割し、書いては validate のループにする（uipath-rpa の incremental 生成方針）。
- 手打ちが困難な長大セレクターは、既存のキャプチャ済み実装から**機械的にコピー**する（Node スクリプト等）。日本語を含む差し替え（クラス名・引数名・既定値）は WriteFile / スクリプト側で行い、PowerShell コマンド本文に日本語を直書きしない。

## 完了ゲート

per-file validate → project build。Object Repository の dangling Reference で validate エラーが出る場合は、**要素レベルの `Reference` / `InformativeScreenshot` 属性を外し**（インラインセレクターは残す）、再 validate する。詳細は `phase3-verify.md`。
