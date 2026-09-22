# Implementation Plan: GoBoard

## Overview

GoBoard を単一の HTML ファイルとして実装する。CSS は `<style>`、JavaScript は `<script>` にインラインで記述し、
ビルド工程・外部ライブラリ・CDN・Web フォントは使用しない。

実装は次の順序で進める。まず DOM 非依存の純粋関数群 `GoBoardLogic`（データモデル定数・座標検証・着手・
勝敗判定・引き分け判定）を構築する。次に自前の軽量 Test_Runner とプロパティ用ジェネレータを用意し、
設計の 8 つの Correctness Property と例ベース単体テストを実装する。最後に UI コントローラを配線し、
ページ読み込み時に「テスト同期実行 → UI 初期化」の順で起動する流れを完成させる。

各着手は不変な `GameState` を返し、無効理由は `OCCUPIED` / `OUT_OF_BOUNDS` / `INVALID_COORDINATE` /
`NOT_IN_PROGRESS` のいずれかで表現する。コード・UI 文字列・コメントでは常に `GoBoard` を用いる。

## Tasks

- [x] 1. HTML スケルトンとデータモデル定数を用意する
  - `index.html` を作成し、`<style>` と `<script>` をインラインで持つ単一ファイル構造を組む
  - `Color` / `CellState` / `GameStatus` を `Object.freeze` の定数として定義する
  - `BOARD_SIZE = 15`、`WIN_LENGTH = 5`、`DIRECTIONS`（横・縦・右下がり斜め・右上がり斜め）を定義する
  - 無効理由コード（`OCCUPIED` / `OUT_OF_BOUNDS` / `INVALID_COORDINATE` / `NOT_IN_PROGRESS`）を定義する
  - _Requirements: 1.1, 3.1, 3.2, 3.3, 3.4_

- [x] 2. GoBoardLogic のゲーム生成と座標検証を実装する
  - [x] 2.1 createGame と nextTurn を実装する
    - `createGame()` が全 225 交点を空、Current_Turn を黒、Game_Status を進行中とする初期状態を返す
    - 既存状態に依存せず常に同一の初期状態を返す（着手済み 0 個）
    - `nextTurn(turn)` が黒⇔白を返す純粋関数を実装する
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 2.4_

  - [x] 2.2 createGame のプロパティテストを書く
    - **Property 8: 新しいゲームの生成は既存状態に依存しない**
    - **Validates: Requirements 1.5**

  - [x] 2.3 isValidCoordinate を実装する
    - 行・列が整数 0〜14 の範囲内かを判定し、範囲外・非整数・null・undefined・NaN・非数値を弾く
    - _Requirements: 3.2, 3.3_

  - [x] 2.4 isValidCoordinate の単体テストを書く
    - 盤内・範囲外・小数・null・undefined・NaN・文字列の各ケースを検証する
    - _Requirements: 3.2, 3.3_

- [x] 3. GoBoardLogic の勝敗判定と引き分け判定を実装する
  - [x] 3.1 checkWin を実装する
    - 着手交点を起点に 4 方向それぞれ両側の同色連続数を数え、`WIN_LENGTH` 以上で true を返す
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [x] 3.2 checkWin の 5 連成立プロパティテストを書く
    - **Property 4: いずれかの方向に 5 連が揃うと対応する色の勝ちになる**
    - **Validates: Requirements 4.1, 4.2, 4.3, 4.4**

  - [x] 3.3 checkWin の複数方向同時成立プロパティテストを書く
    - **Property 5: 複数方向で同時に 5 連が成立しても勝ちは単一の結果になる**
    - **Validates: Requirements 4.5**

  - [x] 3.4 isBoardFull を実装する
    - 全 225 交点が石で埋まっているかを判定する
    - _Requirements: 5.1_

- [x] 4. GoBoardLogic の placeStone を実装して状態遷移を統合する
  - [x] 4.1 placeStone を実装する
    - 座標検証・着手済み検証・進行中検証を行い、無効時は入力状態を変更せず理由付き MoveResult を返す
    - 有効時は不変な新盤面へ石を置き、勝敗・引き分けを判定して Game_Status を更新する
    - 非決着なら nextTurn で手番を切り替える。終局後の着手は `NOT_IN_PROGRESS` で拒否する
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 3.1, 3.2, 3.3, 3.4, 3.5, 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 5.1, 5.3, 5.4, 6.1, 6.2, 6.3, 6.4_

  - [x] 4.2 有効着手の配置プロパティテストを書く
    - **Property 1: 有効着手は手番の色の石を配置する**
    - **Validates: Requirements 2.1, 3.5**

  - [x] 4.3 手番切替プロパティテストを書く
    - **Property 2: 有効かつ非決着の着手は手番を他方へ切り替える**
    - **Validates: Requirements 2.4**

  - [x] 4.4 無効着手プロパティテストを書く
    - **Property 3: 無効な着手は状態を変えず対応する理由を返す**
    - **Validates: Requirements 2.2, 2.3, 3.1, 3.2, 3.3, 3.4, 4.6, 5.3, 5.4, 6.2, 6.3, 6.4**

  - [x] 4.5 既存石の不変性プロパティテストを書く
    - **Property 6: 有効着手は着手した交点以外の既存の石を変更しない**
    - **Validates: Requirements 6.1**

  - [x] 4.6 引き分けプロパティテストを書く
    - **Property 7: 盤が満杯で勝者がいなければ引き分けになる**
    - **Validates: Requirements 5.1**

- [x] 5. Test_Runner を実装してテストを配線する
  - [x] 5.1 test / runTests / assertEqual を実装する
    - `test(name, fn)` で登録、`runTests()` が同期実行し各テストの名前と成否を 1 行ずつ console.log 出力する
    - 総数・成功数・失敗数のサマリを出力し、失敗時はテスト名・期待値・実際の値を出力する
    - いずれのテスト失敗でも例外を外へ伝播させない
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

  - [x] 5.2 プロパティ用ジェネレータを実装する
    - `Math.random` を用いた最小ジェネレータで、進行中 GameState・空交点・範囲外座標・不正座標・
      任意の色/方向/開始位置の 5 連着手列を生成する
    - 各プロパティテストが最低 100 回反復できるようにする
    - _Requirements: 7.1_

  - [x] 5.3 初期化・手番の例ベース単体テストを書く
    - 初期盤面（225 交点が空・着手済み 0 個）、初期手番（黒）、初期ステータス（進行中）を検証する
    - 初手の色が黒、有効着手ごとに黒→白→黒と交互切替することを検証する
    - _Requirements: 7.6, 7.7, 7.8_

  - [x] 5.4 妥当性検証の例ベース単体テストを書く
    - 盤内空交点の受理、既石拒否、範囲外拒否、非整数座標拒否、進行中以外での拒否を検証する
    - _Requirements: 7.9, 7.10, 7.11, 7.12, 7.13_

  - [x] 5.5 勝敗・引き分け・不変性の例ベース単体テストを書く
    - 縦・横・右上がり斜め・右下がり斜めの 5 連勝ち、5 連なし盤面の進行中維持を検証する
    - 満杯かつ無勝者で引き分け、既石への着手で石の色と Current_Turn が不変であることを検証する
    - _Requirements: 7.14, 7.15, 7.16, 7.17, 7.18, 7.19, 7.20_

- [x] 6. チェックポイント - すべてのテストが通ることを確認する
  - Ensure all tests pass, ask the user if questions arise.

- [x] 7. UI コントローラを実装して起動フローを完成させる
  - [x] 7.1 盤面描画とステータス表示を実装する
    - 15×15 の交点を持つ盤面を DOM で描画し、現在の Game_Status（進行中・黒勝ち・白勝ち・引き分け）を表示する
    - _Requirements: 5.2_

  - [x] 7.2 クリックハンドリングと再描画を配線する
    - 交点クリックで placeStone を呼び、返された状態で再描画する
    - 着手拒否時は理由（既に石がある等）をユーザーに提示する
    - _Requirements: 2.2, 5.4, 6.4_

  - [x] 7.3 ページ読み込み時の起動順を配線する
    - ページ読み込み完了時に runTests() を UI 初期化の前に同期実行し、その後ゲームを生成・描画する
    - テスト失敗時も例外で止めず UI 初期化を継続する
    - _Requirements: 7.1, 7.5_

- [x] 8. 最終チェックポイント - すべてのテストが通ることを確認する
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- `*` 付きサブタスクは任意であり、MVP を急ぐ場合はスキップできる。
- 各タスクはトレーサビリティのため対応する要件を参照する。
- プロパティテストは design.md の Correctness Properties を検証し、単体テストは例・エッジケースを補う。
- チェックポイントで段階的に検証する。
- UI 層（DOM 描画・クリック・表示）は自動テストの対象外であり、配線として担保する。
- Task 1 は HTML スケルトンとデータモデル定数を用意するトップレベルタスクで、全サブタスクの前提となる。
  依存グラフには leaf サブタスクのみを含めるため Task 1 は記載していないが、Wave 0 の実行前に完了しておく必要がある。

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "2.3", "3.1", "3.4"] },
    { "id": 1, "tasks": ["2.2", "2.4", "3.2", "3.3", "4.1"] },
    { "id": 2, "tasks": ["4.2", "4.3", "4.4", "4.5", "4.6", "5.1"] },
    { "id": 3, "tasks": ["5.2", "5.3", "5.4", "5.5", "7.1"] },
    { "id": 4, "tasks": ["7.2", "7.3"] }
  ]
}
```
