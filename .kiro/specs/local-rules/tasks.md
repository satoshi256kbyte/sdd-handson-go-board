# Implementation Plan: 連続3が2箇所同時に成立する着手の禁止（Double_Three 禁止）ルール定義の移行

## Overview

`index.html` には既に旧ルール（着手点を通る方向数ベースの連続3の同時成立の判定）が実装され、その単体・
プロパティテストもすべて成功している。本計画は、この旧実装から、改訂後の requirements.md /
design.md が定める新ルール（確定後盤面全体の黒 Open_Three 総数が 2 個以上なら拒否。飛び三は
Open_Three に含めない。離れた 2 個も対象）へ、既存コードを段階的かつ安全に**移行**する。

移行方針は次のとおり。旧関数 `detectOpenThrees` / `isDoubleThreeMove` を廃止し、盤面全体を
走査する `findOpenThrees`（連続 3 かつ延長可能な端を持つもののみ。飛び三は除外）と、
盤面全体の個数で判定する `isDoubleThree(board, color)` へ置き換える。`findThreeInARow` は
`findOpenThrees` へ改名・再定義し、`computeThreeRegistry` はこれを参照する。`applyMove` の
禁止判定は「確定後盤面から 1 回だけ再計算した Three_Registry の `registry.black.length >= 2`」に
変更する。公開 API から旧関数を除去し新関数を追加する。旧セマンティクスに依存する単体テスト・
プロパティテスト・GB_GEN ジェネレータは、新しい 10 個のプロパティ（P1〜P10）と全盤面個数の
意味論に合わせて更新・置換する。

制約は既存どおり維持する。単一 HTML ファイル構成、外部ライブラリ・CDN 不使用、ビジネス
ロジックは DOM 非依存の純粋関数、名称は GoBoard。既存 `placeStone` および既存の基本 32 件の
テストは変更しない。定数 `THREE_LENGTH=3` と `InvalidReason.DOUBLE_THREE` は既に存在するため
そのまま維持する（新規追加タスクは設けない）。

各タスクは前段の成果に積み上がる形で進め、旧関数・旧テストの除去を含めて完了時に dead code や
矛盾するテストが残らないようにする。UI/console.log 配線は既に `applyMove` と Three_Registry を
用いており、Three_Registry の意味が Open_Three に再定義されても文言（Three_In_A_Row）は
成立するため、再構築ではなく検証・微修正タスクとして扱う。

## Tasks

- [x] 1. Open_Three 検出の改名・再定義（旧 detectOpenThrees / findThreeInARow の統合）
  - [x] 1.1 findOpenThrees(board, color) を実装する
    - 盤面全体・4 方向を走査し、指定色がちょうど 3 個「連続」する窓（●●●）のみを候補とする
    - 各候補について前端外側・後端外側の少なくとも一方が盤内かつ空で、その端へ延長して同色 4 連を
      作り得る（延長可能）ものだけを Open_Three とする。両端が相手 Stone・盤端で塞がれた 3 連は除外する
    - 飛び三（●●_●）の判定パスは実装しない（新設計で廃止）
    - 旧 `detectOpenThrees` の「1 本の Line 上の連続 3 ＋ 端の延長可能性」判定ロジックは、
      本関数の内部ヘルパとしてのみ再利用する（着手点を通る方向を返す公開 API としては用いない）
    - 各 Open_Three は構成 3 交点を辞書順昇順に正規化した `key` で一意化し重複排除する
    - 盤面は変更しない
    - _Requirements: 1.1, 1.2, 2.4_

  - [x] 1.2 旧 detectOpenThrees と旧 findThreeInARow を除去する
    - 着手点ベースの公開関数 `detectOpenThrees(board, row, col, color)` を削除する
    - 連続 3 と飛び三の双方を数える旧 `findThreeInARow(board, color)` を削除する
      （その役割は `findOpenThrees` が Open_Three 限定で置き換える）
    - 削除により参照が壊れる箇所を洗い出し、後続タスクで解消する前提で印を付ける
    - _Requirements: 5.1, 5.6_

  - [x] 1.3 findOpenThrees の単体テストを更新・置換する
    - 旧 detectOpenThrees の単体テスト（飛び三検出・方向配列）を削除する
    - `_ X X X _` の連続 3 を端が開いた Open_Three として検出する
    - 相手石・盤端で両端が塞がれた `O X X X O` 形の連続 3 を Open_Three と判定しない
    - 飛び三 `X X _ X` は Open_Three として返さない（旧セマンティクスの反転を明示）
    - _Requirements: 1.1, 1.2_

- [x] 2. Three_Registry を Open_Three のみで再計算する
  - [x] 2.1 computeThreeRegistry(board) を findOpenThrees ベースに再定義する
    - 黒・白それぞれについて `findOpenThrees(board, color)` を呼び、`key` で重複排除した
      `{ black, white }` を新規構築して返す（旧 findThreeInARow への依存を除去する）
    - 盤面を変更せず、既存 Three_Registry も参照せず、確定後盤面に対し毎回同じ結果を返す
    - _Requirements: 2.1, 2.2, 2.4_

  - [x] 2.2 computeThreeRegistry の単体テストを更新する
    - 空盤の Three_Registry が黒・白とも 0 個であることを確認する
    - 同一の連続 3 石を異なる走査経路で数えても 1 件に重複排除されることを確認する
    - 飛び三が Registry に含まれないこと（旧テストの単一飛び 3 期待を削除）を確認する
    - _Requirements: 2.1, 2.4_

- [x] 3. Double_Three 判定を盤面全体の個数へ再定義する
  - [x] 3.1 isDoubleThree(board, color) を実装する
    - `computeThreeRegistry(board)[color].length >= 2` と論理的に同値な、盤面全体の個数判定を実装する
    - 着手点・方向は参照しない。盤面は変更しない
    - _Requirements: 1.1, 1.5_

  - [x] 3.2 旧 isDoubleThreeMove を除去する
    - 方向数 >= 2 で判定する旧 `isDoubleThreeMove(board, row, col, color)` を削除する
    - _Requirements: 5.1, 5.6_

  - [x] 3.3 isDoubleThree の単体テストを更新・置換する
    - 旧 isDoubleThreeMove の単体テスト（縦横 true / 単方向 false）を削除する
    - 盤面全体に黒 Open_Three が 2 個ある盤面で true、1 個以下で false を確認する
    - 離れた 2 個（交わらない）でも true になることを確認する
    - _Requirements: 1.1, 1.5_

- [x] 4. applyMove の禁止判定を Three_Registry の黒個数へ移行する
  - [x] 4.1 applyMove(state, row, col) の禁止ロジックを差し替える
    - 内部で既存 `placeStone` を呼ぶ構造は維持し、無効結果はそのまま伝える（Registry は着手前のまま）
    - 有効かつ着手後 Game_Status が BLACK_WIN なら連続3の同時2箇所成立の禁止を適用せず勝ちを優先する
    - 有効かつ黒の非勝ち着手のとき、確定後盤面から Three_Registry を 1 回だけ再計算し、
      `registry.black.length >= 2` なら valid=false・reason=DOUBLE_THREE に差し替え、state を
      着手前へ、Three_Registry も着手前の内容へ戻して返す（旧 isDoubleThreeMove 呼び出しを撤去）
    - 白の着手・黒で黒 Open_Three が 1 個以下の手は有効のまま受理する
    - 有効確定時は再計算した Three_Registry を LocalMoveResult に含める
    - 既存 `placeStone` は変更しない
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 2.2, 2.3, 4.1, 5.1, 5.2_

  - [x] 4.2 applyMove の単体テストを更新・置換する
    - 着手点で縦横 2 方向の黒 Open_Three が同時成立する黒着手を DOUBLE_THREE で拒否し、
      盤面・Current_Turn が着手前と同一であることを確認する（既存ケースを新意味論に維持）
    - 既存の黒 Open_Three 1 個に加え、離れた位置に 2 個目を作る黒着手を DOUBLE_THREE で拒否する
      ケースを追加する（新規：全盤面個数の意味論）
    - 旧「単一方向 Open_Three は受理」ケースを「着手後の黒 Open_Three が 1 個以下となる手は受理」に
      作り替える
    - 白の Double_Three（白 Open_Three 2 個以上）着手を受理し石を配置することを確認する
    - 黒の連続3が2箇所成立し、かつ 5 連の着手で BLACK_WIN になることを確認する
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 4.1_

- [x] 5. 公開 API エクスポートの更新
  - [x] 5.1 GoBoardLogic の公開 API を新関数へ差し替える
    - 公開リストから `detectOpenThrees` と `isDoubleThreeMove` を除去する
    - 公開リストへ `findOpenThrees` と `isDoubleThree` を追加する
    - `computeThreeRegistry` と `applyMove` は引き続き公開する
    - 旧名を参照している内部・テストの残存箇所がないことを確認する
    - _Requirements: 5.1, 5.2, 5.3_

- [x] 6. Checkpoint - ロジック移行後の検証
  - Ensure all tests pass, ask the user if questions arise.
  - findOpenThrees / computeThreeRegistry / isDoubleThree / applyMove の更新済み単体テストが通り、
    旧 detectOpenThrees / isDoubleThreeMove / findThreeInARow への参照が残っていないことを確認する

- [ ] 7. GB_GEN ジェネレータを新意味論へ更新する
  - [x] 7.1 連続3が2箇所成立する系のジェネレータを全盤面個数ベースへ再定義する
    - 旧 `randomBlackDoubleThree`（着手点十字）を、黒着手で盤面全体の黒 Open_Three が 2 個以上と
      なる盤面・着手を生成する形へ更新する（十字で 2 個成立させる形を含む）
    - 旧 `randomBlackSingleOpenThree` を、黒着手後の盤面全体の黒 Open_Three が 1 個以下に収まる
      盤面・着手を生成する形（受理される手）へ更新する
    - 旧 `randomWhiteDoubleThree` を、白着手で盤面全体の白 Open_Three が 2 個以上となる盤面
      （手番を白に設定）へ更新する
    - 旧 `randomBlackDoubleThreeWin` を、黒 Open_Three 2 個以上かつ着手が 5 連にもなる盤面
      （Win_Condition と連続3の2箇所成立が同時成立）へ更新する
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

  - [x] 7.2 「既存 Open_Three ＋ 離れた 2 個目」ジェネレータを追加する
    - 着手点と無関係な黒 Open_Three を 1 個先置きし、今回の黒着手が離れた位置に 2 個目の黒
      Open_Three を成立させる（互いに交わらない）盤面・着手を生成する
    - あわせて「黒着手後も盤面全体の黒 Open_Three を 1 個以下に保つ」受理ケース用の生成も用意する
    - _Requirements: 1.5, 1.2_

- [x] 8. プロパティテストを新 P1〜P10 へ再定義する
  - [x] 8.1 旧プロパティテストを新定義へ差し替える
    - `Feature: local-rules, Property N` タグの旧プロパティテスト（旧セマンティクス）を削除する
    - 特に旧「Property 8: detectOpenThrees は相異なる方向を返す」を撤去する
    - 以降 8.2〜8.11 で新 P1〜P10 を登録する土台を整える
    - _Requirements: 5.6_

  - [x] 8.2 Property 1 のプロパティベーステストを実装する
    - **Property 1: 黒の連続3が2箇所成立する着手は無効となり状態を変えない**
    - **Validates: Requirements 1.1, 4.1**
    - タグ: `Feature: local-rules, Property 1: ...`、最低 100 回反復

  - [x] 8.3 Property 2 のプロパティベーステストを実装する
    - **Property 2: 黒 Open_Three が 1 個以下となる着手は受理される**
    - **Validates: Requirements 1.2**
    - タグ: `Feature: local-rules, Property 2: ...`、最低 100 回反復

  - [x] 8.4 Property 3 のプロパティベーステストを実装する
    - **Property 3: 白の連続3が2箇所成立する着手は受理される**
    - **Validates: Requirements 1.3**
    - タグ: `Feature: local-rules, Property 3: ...`、最低 100 回反復

  - [x] 8.5 Property 4 のプロパティベーステストを実装する
    - **Property 4: Win_Condition は連続3の同時2箇所成立の禁止より優先される**
    - **Validates: Requirements 1.4**
    - タグ: `Feature: local-rules, Property 4: ...`、最低 100 回反復

  - [x] 8.6 Property 5 のプロパティベーステストを実装する
    - **Property 5: 有効着手後の Three_Registry は確定後盤面と一致する**
    - **Validates: Requirements 2.2**
    - タグ: `Feature: local-rules, Property 5: ...`、最低 100 回反復

  - [x] 8.7 Property 6 のプロパティベーステストを実装する
    - **Property 6: 無効着手では Three_Registry が変化しない**
    - **Validates: Requirements 2.3, 3.4**
    - タグ: `Feature: local-rules, Property 6: ...`、最低 100 回反復

  - [x] 8.8 Property 7 のプロパティベーステストを実装する
    - **Property 7: Three_Registry は色ごとに分離され重複を持たない**
    - **Validates: Requirements 2.4**
    - タグ: `Feature: local-rules, Property 7: ...`、最低 100 回反復

  - [x] 8.9 Property 8 のプロパティベーステストを実装する
    - **Property 8: 検出される並びは常に延長可能な連続 3 の Open_Three である**
    - **Validates: Requirements 1.1, 2.2**
    - タグ: `Feature: local-rules, Property 8: ...`、最低 100 回反復
    - 旧 P8（方向を返す）を置換し、飛び三・両端閉塞が返されないことを検証する

  - [x] 8.10 Property 9 のプロパティベーステストを実装する
    - **Property 9: 黒の連続3が2箇所成立する着手以外では applyMove は placeStone と同一の可否・状態を返す**
    - **Validates: Requirements 5.2**
    - タグ: `Feature: local-rules, Property 9: ...`、最低 100 回反復

  - [x] 8.11 Property 10 のプロパティベーステストを実装する
    - **Property 10: 禁止判定は Three_Registry の黒個数と一致する（唯一の情報源）**
    - **Validates: Requirements 1.1, 3.3**
    - タグ: `Feature: local-rules, Property 10: ...`、最低 100 回反復
    - DOUBLE_THREE 拒否と `computeThreeRegistry(board).black.length >= 2` が同値であることを検証する

- [x] 9. UI/ログ配線の検証と微調整（DOM 依存・自動テスト対象外）
  - [x] 9.1 Three_Registry 再定義後のログ文言と個数の整合を検証する
    - クリック処理が `applyMove` を呼び、Three_Registry から黒・白の Three_In_A_Row の有無・個数を
      console.log 出力している既存配線を確認する
    - Registry が Open_Three に再定義されたため、ログの Three_In_A_Row 表記が Open_Three の意味と
      一致し、出力個数が禁止判定の基準（黒個数 >= 2）と一致することを確認する
    - 不一致があれば文言・参照先のみを最小限に調整する（再構築はしない）。DOUBLE_THREE 通知が
      他の拒否理由と区別できることも確認する
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 4.2, 4.3, 4.4, 5.3_

- [x] 10. Final checkpoint - 移行完了と回帰確認
  - Ensure all tests pass, ask the user if questions arise.
  - 既存の基本 32 件のテストが引き続きすべて成功し、失敗数が増えないことを確認する
  - 更新後の local-rules 単体テストおよび新プロパティテスト（P1〜P10）がすべて成功することを確認する
  - 旧関数（detectOpenThrees / isDoubleThreeMove / findThreeInARow）と旧テスト・旧ジェネレータが
    残存せず、dead code や矛盾するテストがないことを確認する
  - ブラウザでの手動サニティチェック: ユーザーが遭遇した実シナリオ（黒の着手で 2 個目の盤面全体
    Open_Three が成立する状況、例として横 3 個連続＋縦 3 個連続のスクリーンショット状況）が連続3の2箇所成立の通知とともに
    拒否され、ログの Open_Three 個数が禁止判定基準と一致することを確認する
  - _Requirements: 5.5, 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

## Notes

- `*` が付いたサブタスクは任意（テスト関連）だが、本移行はビジネスロジックの正当性が新プロパティで
  規定されるため、移行完了前にテストを通すことを強く推奨する。
- タスク 1.2 / 3.2 / 8.1 は旧コード・旧テストの明示的な除去であり、dead code と矛盾するテストを
  残さないためコア作業として扱う（`*` を付けない）。
- タスク 9（UI 配線・DOM・console.log）はステアリング方針により自動テスト対象外。Three_Registry の
  意味が変わっても配線構造は再利用できるため、再構築ではなく検証・微調整に留める。
- 既存 `placeStone` および既存の基本 32 件のテストは変更しない。定数 `THREE_LENGTH` と
  `InvalidReason.DOUBLE_THREE` は既存のまま維持する。
- 各タスクはトレーサビリティのため対応要件を参照し、プロパティタスクは設計の Property 番号・
  Validates 行・タグ・反復回数を明示する。

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "1.3"] },
    { "id": 2, "tasks": ["2.1", "3.1"] },
    { "id": 3, "tasks": ["2.2", "3.2", "3.3"] },
    { "id": 4, "tasks": ["4.1"] },
    { "id": 5, "tasks": ["4.2", "5.1"] },
    { "id": 6, "tasks": ["7.1", "7.2", "8.1"] },
    { "id": 7, "tasks": ["8.2", "8.3", "8.4", "8.5", "8.6", "8.7", "8.8", "8.9", "8.10", "8.11"] },
    { "id": 8, "tasks": ["9.1"] }
  ]
}
```
