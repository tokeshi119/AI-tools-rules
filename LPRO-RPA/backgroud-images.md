2025-12-18

# 写真編集モーダル待ちタイムアウト問題 — 対応まとめ

## 1. 事象（発生していたエラー）
- Playwright が `写真を編集` モーダルの見出しを待つ処理でタイムアウト
- 具体的には `locator.waitFor` が `Timeout 10000ms exceeded` になって RPA が失敗扱いで終了

## 2. 原因の整理（何が起きていたか）
このRPA実装は「モーダルを開く命令」を出しているのではなく、
**画像アップロード後に LINE 側で自動表示されるはずのモーダルを“待っているだけ”**という構造。

そのため、以下のいずれかが起きるとタイムアウトになり得る：
- (A) モーダルが表示されない（LINE側遅延・JS不調・別ダイアログ等）
- (B) 表示が遅く、10秒（`ELEMENT_TIMEOUT`）を超える
- (C) 見た目は出ていても、`getByRole(heading, name="写真を編集")` に一致しないDOM構造になっている
- (補足) 配布先PCの負荷（Teams/Office等）で描画・JS実行・ネットワークが遅くなり、表示が遅延する可能性もある

## 3. RPAのリトライ挙動（失敗時の流れ）
- `waitForEditPictureModal()` がタイムアウトすると例外が発生
- `createLineOfficialAccount*` 側のリトライループで捕捉され、スクショ取得などをしつつ再実行
- 最大試行回数は設定上「最初の実行 + 最大2回リトライ」＝ **最大3回**

## 4. 検証結果
- 待ち時間を延ばして検証したところ、**100件連続で問題なく完了**
- これにより「モーダルが出るが遅い」ケースが主因だった可能性が高い

## 5. 実装変更（今回の対応内容）
### 5.1 目的
- グローバルな `ELEMENT_TIMEOUT` を変えずに、
  **写真編集モーダル待ちだけ**十分長いタイムアウトを確保する
- `PROFILE_UPLOAD_TIMEOUT` の値変更が他処理へ波及するのを防ぐ

### 5.2 変更点
- `WAIT_CONFIG` に写真編集モーダル専用タイムアウトを追加
  - `WAIT_CONFIG.EDIT_PICTURE_MODAL_TIMEOUT = 50000`
- `AccountPageProfileEditPictureModal.waitForEditPictureModal()` のデフォルト待機を
  - `WAIT_CONFIG.ELEMENT_TIMEOUT(10000)` → `WAIT_CONFIG.EDIT_PICTURE_MODAL_TIMEOUT(50000)` に変更
- 一時的に追加していた `try/catch` + `console.warn` の診断ログは、
  検証完了により **不要** と判断して削除

### 5.3 `PROFILE_UPLOAD_TIMEOUT` の扱い
- 検証のため一時的に `PROFILE_UPLOAD_TIMEOUT` を 50000 に変更していたが、
  これは **他箇所にも利用されているため影響が大きい**
- 今回は `PROFILE_UPLOAD_TIMEOUT` を **20000 に戻し**、
  写真編集モーダル待ちにだけ専用の `EDIT_PICTURE_MODAL_TIMEOUT` を割り当てる構成にした

## 6. 影響範囲（どこに効くか）
### 6.1 今回の修正が効く範囲
- `waitForEditPictureModal()` を呼んでいる全てのフローで
  **写真編集モーダル待ちが最大 50 秒**になる

### 6.2 `PROFILE_UPLOAD_TIMEOUT` が影響し得る箇所（調査結果）
`WAIT_CONFIG.PROFILE_UPLOAD_TIMEOUT` は主に「公開しました」等のツールチップ待ちで使われており、
値を 50000 にすると「失敗検知が遅くなる」副作用が出る可能性がある。

該当例：
- `waitAndClearForToolTip(..., WAIT_CONFIG.PROFILE_UPLOAD_TIMEOUT)` を使っている処理群

## 7. 言語別（JP/TH/KR/EN）への反映について
- 各言語の `createLineOfficialAccount*.ts` は分かれているが、
  プロフィール画面操作は共通のページオブジェクト（JP配下）を使っている
- そのため、今回の変更は **JP/TH/KR/EN 全てに自動反映**される
  - 個別の言語ファイルを同じように修正する必要はない

## 8. コミットメッセージ案（日本語）
- `写真編集モーダル待機のタイムアウト調整（専用WAIT_CONFIG追加）`

### 9.追加
- 余裕を持ってTIMEOUTを50000msから60000msへ変更


## 10.結果
どうやらソリュ側でVPNを新しいものに変えたら改善した模様
[Backlog](https://svsprj.backlog.jp/view/LINESPE-724#comment-133783909)