# Implementation Plan: 三三の禁止（Double_Three 禁止）ローカルルール

## Overview

既存の GoBoard（単一 HTML ファイル `index.html`）に、Black_Player の Double_Three 着手を
禁止するローカルルールを追加する。実装はすべて `index.html` 内で完結させ、新規ファイルや
モジュール分割は行わない。追加のビジネスロジックは DOM 非依存の純粋関数として `GoBoardLogic`
に追加し、既存の `placeStone` は変更しない。三三判定・Three_Registry 再計算・`applyMove` ラッパは
純粋関数として実装してプロパティベーステスト・単体テストで検証し、UI 配線・通知表示・
console.log 出力は表示/入力層に閉じる（自動テスト対象外）。

各タスクは前段の成果に積み上がる形で進め、最終的に UI 配線で全体を統合する。既存テストは
一切変更せず、追加後も引き続き成功させる。

## Tasks

- [ ] 1. 定数・理由コードの追加（additive）
  - `index.html` のビジネスロジック層に `THREE_LENGTH = 3` 定数を追加する
  - 既存 `InvalidReason` に `DOUBLE_THREE: 'DOUBLE_THREE'` を追加する（既存値は変更しない）
  - 既存の `Color` / `CellState` / `GameStatus` / `Board` / `GameState` / `DIRECTIONS` は変更しない
  - _Requirements: 1.1, 4.2, 5.1, 5.2_

- [ ] 2. Open_Three 検出の実装
  - [ ] 2.1 detectOpenThrees(board, row, col, color) を実装する
    - 着手交点を含む長さ 3〜4 の窓を 4 方向それぞれについて列挙し、連続 3（contiguous-3）と
      単一飛び 3（single-gap-3）の並びを候補として抽出する
    - 各候補について両端の延長可能性（相手石・盤端に妨げられず同色 4 連を作り得るか）を判定し、
      Open_Three 成立方向を相異なる Line 方向の配列として返す
    - 盤面は変更せず、同一方向は 1 回として数える
    - _Requirements: 1.1, 1.2_

  - [ ]* 2.2 detectOpenThrees の単体テストを追加する
    - `_ X X X _` の連続 3 で 1 方向の Open_Three を検出する
    - `X X _ X` の単一飛び 3 で Open_Three を検出する
    - 盤端・相手石で両端が塞がれた 3 連は Open_Three と判定しない
    - _Requirements: 1.1, 1.2_

- [ ] 3. Double_Three 判定の実装
  - [ ] 3.1 isDoubleThreeMove(board, row, col, color) を実装する
    - detectOpenThrees の返す成立方向集合の要素数が 2 以上なら true を返す
    - 盤面は変更しない
    - _Requirements: 1.1_

  - [ ]* 3.2 isDoubleThreeMove の単体テストを追加する
    - 縦横 2 方向で同時に Open_Three となる黒着手で true を返す
    - 1 方向のみ Open_Three となる着手で false を返す
    - _Requirements: 1.1, 1.2_

- [ ] 4. Three_Registry の再計算実装
  - [ ] 4.1 findThreeInARow(board, color) と computeThreeRegistry(board) を実装する
    - findThreeInARow は盤面全体を走査し、指定色の Three_In_A_Row（連続 3・単一飛び 3）を列挙する
    - 各 Three_In_A_Row は構成 3 交点を辞書順昇順に正規化した `key` で一意化し重複排除する
    - computeThreeRegistry は黒・白を区別した `{ black, white }` を新規構築して返し、盤面を変更しない
    - _Requirements: 2.1, 2.2, 2.4_

  - [ ]* 4.2 findThreeInARow / computeThreeRegistry の単体テストを追加する
    - 空盤の Three_Registry が黒・白とも 0 個であることを確認する
    - 同一 3 石を異なる走査経路で数えても 1 件に重複排除されることを確認する
    - _Requirements: 2.1, 2.4_

- [ ] 5. Checkpoint - ここまでの検出・登録ロジックのテストを通す
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. applyMove ラッパの実装と公開 API 追加
  - [ ] 6.1 applyMove(state, row, col) を実装する
    - 内部で既存 `placeStone` を呼び、無効結果はそのまま伝える（Three_Registry は着手前のまま）
    - 有効結果に対し、着手後 Game_Status が BLACK_WIN なら三三禁止を適用せず勝ちを優先する
    - 着手前手番が Black_Player かつ isDoubleThreeMove が true なら valid=false・reason=DOUBLE_THREE に
      差し替え、状態を着手前の state に戻して返す
    - 白の着手・黒の単方向・三を作らない手は有効のまま受理する
    - 有効確定時は computeThreeRegistry で Three_Registry を再計算し LocalMoveResult に含める
    - 既存 `placeStone` は変更しない
    - detectOpenThrees / isDoubleThreeMove / findThreeInARow / computeThreeRegistry / applyMove を
      `GoBoardLogic` の公開 API に追加する
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.2, 2.3, 4.1, 5.1, 5.2, 5.3_

  - [ ]* 6.2 applyMove の単体テストを追加する
    - 典型的な黒三三着手を DOUBLE_THREE で拒否し、盤面・Current_Turn が着手前と同一であることを確認する
    - 黒の単方向 Open_Three 着手を受理し石を配置することを確認する
    - 白の Double_Three 着手を受理し石を配置することを確認する
    - 黒の三三かつ 5 連の着手で BLACK_WIN になることを確認する
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 4.1_

- [ ] 7. プロパティベーステストの追加
  - [ ]* 7.1 GB_GEN に本機能用ジェネレータを追加する
    - 黒の Double_Three を確実に構成する盤面と着手を生成する
    - 黒の単方向 Open_Three のみを構成する盤面と着手を生成する
    - 白の Double_Three を構成する盤面（手番を白に設定）を生成する
    - 黒の Double_Three かつ 5 連にもなる盤面（Win_Condition と三三が同時成立）を生成する
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

  - [ ]* 7.2 Property 1 のプロパティベーステストを実装する
    - **Property 1: 黒の三三着手は無効となり状態を変えない**
    - **Validates: Requirements 1.1, 4.1**
    - タグ: `Feature: local-rules, Property 1: ...`、最低 100 回反復

  - [ ]* 7.3 Property 2 のプロパティベーステストを実装する
    - **Property 2: 黒の単方向 Open_Three の着手は受理される**
    - **Validates: Requirements 1.2**
    - タグ: `Feature: local-rules, Property 2: ...`、最低 100 回反復

  - [ ]* 7.4 Property 3 のプロパティベーステストを実装する
    - **Property 3: 白の三三着手は受理される**
    - **Validates: Requirements 1.3**
    - タグ: `Feature: local-rules, Property 3: ...`、最低 100 回反復

  - [ ]* 7.5 Property 4 のプロパティベーステストを実装する
    - **Property 4: Win_Condition は三三禁止より優先される**
    - **Validates: Requirements 1.4**
    - タグ: `Feature: local-rules, Property 4: ...`、最低 100 回反復

  - [ ]* 7.6 Property 5 のプロパティベーステストを実装する
    - **Property 5: 有効着手後の Three_Registry は確定後盤面と一致する**
    - **Validates: Requirements 2.2**
    - タグ: `Feature: local-rules, Property 5: ...`、最低 100 回反復

  - [ ]* 7.7 Property 6 のプロパティベーステストを実装する
    - **Property 6: 無効着手では Three_Registry が変化しない**
    - **Validates: Requirements 2.3, 3.4**
    - タグ: `Feature: local-rules, Property 6: ...`、最低 100 回反復

  - [ ]* 7.8 Property 7 のプロパティベーステストを実装する
    - **Property 7: Three_Registry は色ごとに分離され重複を持たない**
    - **Validates: Requirements 2.4**
    - タグ: `Feature: local-rules, Property 7: ...`、最低 100 回反復

  - [ ]* 7.9 Property 8 のプロパティベーステストを実装する
    - **Property 8: detectOpenThrees は相異なる方向を返す**
    - **Validates: Requirements 1.1**
    - タグ: `Feature: local-rules, Property 8: ...`、最低 100 回反復

  - [ ]* 7.10 Property 9 のプロパティベーステストを実装する
    - **Property 9: 非・黒三三の着手では applyMove は placeStone と同一の可否・状態を返す**
    - **Validates: Requirements 5.2**
    - タグ: `Feature: local-rules, Property 9: ...`、最低 100 回反復

- [ ] 8. Checkpoint - 追加テストと既存テストがすべて通ることを確認する
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. UI コントローラへの配線（DOM 依存・自動テスト対象外）
  - [ ] 9.1 クリック処理を applyMove 呼び出しへ切り替える
    - クリック時に `placeStone` ではなく `applyMove(state, row, col)` を呼ぶ
    - 起動時に `computeThreeRegistry(createGame().board)` で初期 Three_Registry を構築する
    - 有効着手時は盤面・ステータスを再描画し、通知領域 `#gb-notice` をクリアする
    - _Requirements: 4.4, 5.3_

  - [ ] 9.2 拒否通知と Three_In_A_Row ログ出力を配線する
    - 無効着手時に reason に応じたメッセージを `#gb-notice` に提示し、`DOUBLE_THREE` は
      他の拒否理由（既に石がある、対局終了済み）と区別できる「三三禁止により着手できない」旨とする
    - 有効着手確定時に Three_Registry を用いて黒・白それぞれの Three_In_A_Row の有無を
      console.log に 1 行ずつ出力し、1 つ以上存在する色は個数も含める
    - 無効着手時は Three_In_A_Row の有無ログを出力しない
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 4.2, 4.3_

- [ ] 10. Final checkpoint - 全体統合と回帰確認
  - Ensure all tests pass, ask the user if questions arise.
  - 追加テスト（Property 1〜9・単体テスト）と既存の全テストが引き続き成功し、失敗数が増えないことを確認する
  - _Requirements: 5.5, 6.1, 6.2, 6.3, 6.4, 6.5_

## Notes

- `*` が付いたサブタスクは任意（テスト関連）であり、MVP を急ぐ場合はスキップできる。ただし
  本機能はビジネスロジックの正当性がプロパティで規定されるため、実装完了前にテストを通すことを推奨する。
- タスク 9（UI 配線・DOM・console.log）はステアリング方針により自動テストの対象外とし、`*` を
  付けていないコアな配線作業として扱う。ビジネスロジックは DOM 非依存の純粋関数として保つ。
- 各タスクはトレーサビリティのため対応要件を参照している。
- チェックポイントで段階的に検証し、既存 `placeStone` および既存テストを変更しない。

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["2.2", "3.1", "4.1"] },
    { "id": 3, "tasks": ["3.2", "4.2"] },
    { "id": 4, "tasks": ["6.1"] },
    { "id": 5, "tasks": ["6.2", "7.1"] },
    { "id": 6, "tasks": ["7.2", "7.3", "7.4", "7.5", "7.6", "7.7", "7.8", "7.9", "7.10"] },
    { "id": 7, "tasks": ["9.1"] },
    { "id": 8, "tasks": ["9.2"] }
  ]
}
```
