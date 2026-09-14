# フェーズ3詳細：検証（validate → build → run）と落とし穴

## 二段階検証（uipath-rpa Rule 3）

各フェーズの末尾で必ず実施する。

1. **per-file validate**（作成・編集のたび、エラー0まで）：
   ```
   uip rpa validate --file-path "<FILE>" --project-dir "<DIR>" --output json
   ```
   → `"message": "No diagnostics found."` を確認。
2. **project build**（全ファイルの validate が通ってから、完了宣言の前に）：
   ```
   uip rpa build "<DIR>" --output json
   ```
   → `"Result": "Success"` / `"Success": true` を確認。

- validate は通るが build で落ちる典型：不明メンバ名（例：NGetText の出力は `<uix:NGetText.Text>`／`OutArgument<String>`。`Value` は存在しない）、無効な enum。**build まで通して初めて「検証済み」**。
- 5回上限のループ、1回1原因の修正（Rule 3）。

## run による実機確認（フェーズ3の本体）

```
uip rpa run "<FILE>" --project-dir "<DIR>" --output json
```
- 成否は **外側の `Result` / `HasErrors`** で判定する。ログ項目の `Level`（Error/Warning）では判定しない（**Rule 8a**）。正常なワークフローでも観測用に Error レベルのログを出すことがある。
- UIA は実行時にブラウザー／アプリ操作が走り、実機の画面状態に依存する。

## セレクター不一致（実機差異）

- 実行時にターゲットが見つからない場合、対象アプリの現在の画面に合わせて `uia-configure-target` で**再キャプチャ**して調整する（uia-starter-guide の runtime selector recovery）。
- 手書きで直そうとしない。

## エンコーディングの落とし穴（実績あり）

- **日本語文字列を PowerShell のコマンド本文に直書きしない**。送信時に文字化けする。日本語コンテンツ（XAML の要素テキスト等）は必ず **WriteFile（UTF-8）** で書く。
- CLI に日本語を含むパスを渡すとき：対象フォルダへ `Set-Location -LiteralPath "<日本語パス>"` してから、ネイティブコマンドには **ASCII の相対パス**（`"TestCase.xaml"` / `"."`）を渡すと安定する。JSON 出力のため先頭で `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`。
- どうしても引数に日本語パスが必要なら、文字コード単位で組み立てる（`[char]0x30D7` 等）方法もあるが、上の `Set-Location` 方式が簡単。

## 書き込み先の制約（このハーネス固有）

- **WriteFile はプロジェクトフォルダ内に限定**される。`.autopilot/skills/` 等プロジェクト外へは WriteFile できない（`Path is not located in the project folder`）。
- プロジェクト外へファイルを置くには：まずプロジェクト内へ WriteFile（UTF-8 正しく）→ PowerShell の `Copy-Item` で**バイト保持コピー**する（コピーは再エンコードしないので日本語が壊れない）。

## 引数バインド

- 引数（例：`メールアドレス`、既定 `demo@demo.com`）は `x:Members` に定義し、ルート属性で既定値を与える。
- UIA 版で参照するときは `Text="[メールアドレス]"` のように角括弧。`Text=` プレフィックスで引数の既定値定義と区別される。
- データ駆動テストにする場合は `dataVariationFilePath` を `fileInfoCollection` に足す（uipath-rpa の testing-guide）。

## Object Repository の dangling Reference

- UIA 実装を流用した際、`.objects` に無い要素レベル References が残ると validate エラーになることがある。
- その場合は**要素レベルの `Reference` / `InformativeScreenshot` 属性のみ外し**、インラインセレクター（Fuzzy/Full/Scope）は残して再 validate する。

## 非ブロッキング警告（Success を妨げない代表例）

- Automation Hub URL 未定義（組織ガバナンス）
- Log Message 未使用 / Comment 既定名・重複
- セレクターの `idx` 属性、クリックの verification 未有効
- 引数名が命名規則（`in_` プレフィックス）に非準拠 — 日本語引数名だと出るが動作に影響なし

これらはエラーではないので完了を妨げない。必要なら後で整える。

## 実行例ログ（validate / build / run）

以下はこのプロジェクト（`TestCase.xaml`, UIA 版）での出力例。すべて `--output json` 前提。`[...]` はプロジェクト名。

### validate（per-file）— 成功【実出力】

```json
{
  "Result": "Success",
  "Code": "ToolResult",
  "Data": { "message": "No diagnostics found." }
}
```
→ `Data.message` が `No diagnostics found.` なら診断エラー0。

### build（project）— 成功【実出力】

```json
{
  "Result": "Success",
  "Code": "Build",
  "Data": { "Success": true }
}
```
成功時も stderr に**非ブロッキング警告**が出る（実出力そのまま）：
```
[WARN] [Tests] [...] Comment display name is defined many times. Current allowed threshold is 1. Code contains 2.
[WARN] [Tests] [...] Your organization requires your project to have an Automation Hub URL defined.
[WARN] [Tests] [...] Target > Strict selector contains non allowed attribute idx='2' in activity テキストを取得 'You've been approved'
[WARN] [Tests] [...] Activity Comment has a default name.
[WARN] [Tests] [...] The following UI Automaton activity クリック 'Submit Loan Application' does not have the verification feature enabled.
[WARN] [Tests] [...] Argument メールアドレス does not respect the set pattern ^in_(dt_)?([A-Z]|[a-z])+([0-9])*$
```
→ これらは WARN であってエラーではない。**合否は `Result: "Success"` / `Data.Success: true`** で判定する。

### run（実機実行）— 出力の読み方＋代表例

```
uip rpa run "TestCase.xaml" --project-dir "." --output json
```
**成否は外側の `Result` と `Data.HasErrors` で判定する**（Rule 8a。ログ項目の `Level` では判定しない — 正常でも観測用に Error レベルのログを出すことがある）。

代表的な成功出力（形の例）：
```json
{
  "Result": "Success",
  "Code": "Run",
  "Data": { "HasErrors": false }
}
```
失敗時は `Result` が非 Success かつ `Data.HasErrors: true`。UIA は実機ブラウザーを操作するため、セレクター不一致・画面差異はここで顕在化する（→ `uia-configure-target` で再キャプチャして調整）。

> 注：validate / build は本プロジェクトの**実出力**。run は**代表例**（形の説明用）で、実際の出力は実行環境・結果により変わる。実ログを採取するには上記 `uip rpa run` を実行する。
