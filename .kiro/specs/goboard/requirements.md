# Requirements Document

## Introduction

GoBoard は、15×15 の格子盤上で 2 人のプレイヤーが交互に石を置き、
縦・横・斜めのいずれかの方向に同じ色の石を 5 つ連続で並べたプレイヤーが勝利するボードゲームである。

先手は黒石、後手は白石を使用する。一度石を置いた交点は変更できない。
どちらのプレイヤーも 5 連を達成しないまま盤面のすべての交点が埋まった場合は引き分けとなる。

本機能は HTML + CSS + JavaScript を 1 ファイルで完結させる形で実装する。
盤面状態の管理・勝敗判定・手番制御といった UI に依存しないビジネスロジックは、
DOM に依存しない純粋なモジュールとして切り出し、ページ読み込み時に自動テストを実行する。

## Glossary

- **GoBoard**: 本ゲーム全体、およびゲーム状態を管理するビジネスロジックのシステム名。
- **Board**: 15×15 の交点からなる格子盤。各交点は「空」「黒」「白」のいずれかの状態を持つ。
- **Intersection**: 盤面を構成する縦線と横線が交わる点。
  石はマス目の中ではなく、この交点の上に置かれる。
  盤全体では 15×15 = 225 個の Intersection が存在し、各交点は「空」「黒」「白」のいずれかの状態を持つ。
  各 Intersection は行番号 row（0〜14）と列番号 col（0〜14）の組で一意に識別される。
- **Stone**: 交点に置かれる石。色は黒または白。
- **Black_Player**: 先手のプレイヤー。黒石を使用する。
- **White_Player**: 後手のプレイヤー。白石を使用する。
- **Current_Turn**: 現在手番を持つプレイヤーを表す状態（黒または白）。
- **Line**: 縦・横・右上がり斜め・右下がり斜めのいずれかの直線方向。
- **Win_Condition**: 同一の色の Stone が同一の Line 上に 5 つ連続で並んだ状態。
- **Game_Status**: ゲームの進行状態。「進行中」「黒勝ち」「白勝ち」「引き分け」のいずれか。
- **Test_Runner**: ページ読み込み時にビジネスロジックの自動テストを実行し、結果を出力する仕組み。

## Requirements

### Requirement 1: 盤面の初期化

**User Story:** プレイヤーとして、ゲーム開始時に空の盤面が用意されてほしい。
そうすれば最初の一手から対局を始められる。

#### Acceptance Criteria

1. WHEN 新しいゲームが開始される, THE GoBoard SHALL 15 行 15 列（合計 225 個）すべての Intersection を「空」の状態で初期化する
2. WHEN 新しいゲームが開始される, THE GoBoard SHALL Current_Turn を Black_Player に設定する
3. WHEN 新しいゲームが開始される, THE GoBoard SHALL Game_Status を「進行中」に設定する
4. WHEN 新しいゲームが開始される, THE GoBoard SHALL 直前のゲームで各 Intersection に配置されていたすべての石を除去し、着手済みの Intersection が 0 個の状態にする
5. IF 進行中のゲームが存在する状態で新しいゲームの開始が要求される, THEN THE GoBoard SHALL 現在の盤面状態・Current_Turn・Game_Status をすべて破棄し、初期状態で置き換える

### Requirement 2: 交互の着手

**User Story:** プレイヤーとして、黒と白が交互に石を置けるようにしてほしい。
そうすれば公平にゲームを進められる。

#### Acceptance Criteria

1. WHEN 空の Intersection に着手が行われる, THE GoBoard SHALL その Intersection に Current_Turn の色の Stone を配置する
2. IF 既に Stone が配置されている Intersection に着手が行われる, THEN THE GoBoard SHALL 着手を拒否し、盤面状態と Current_Turn を変更前のまま保持し、当該 Intersection が着手不可であることを示す表示を行う
3. IF Win_Condition または引き分けが既に成立してゲームが終了している状態で着手が行われる, THEN THE GoBoard SHALL 着手を拒否し、盤面状態と Current_Turn を変更前のまま保持する
4. WHEN 着手が正常に完了し、かつ Win_Condition と引き分けのいずれも成立しない, THE GoBoard SHALL Current_Turn を、直前に着手したプレイヤーとは異なる色（Black_Player と White_Player の二者のうち他方）のプレイヤーに切り替える
5. WHERE ゲーム開始直後の最初の着手, THE GoBoard SHALL Current_Turn を Black_Player とし、Black_Player の色の Stone を配置する

### Requirement 3: 着手の妥当性検証

**User Story:** プレイヤーとして、すでに石がある場所や盤外への着手を防いでほしい。
そうすれば不正な操作でゲームが壊れない。

#### Acceptance Criteria

1. IF すでに Stone が置かれている Intersection に着手が行われる, THEN THE GoBoard SHALL 盤面と Current_Turn を変更せず、着手が無効である旨と無効理由（既に石がある）を示す結果を返す
2. IF 行番号または列番号のいずれかが整数 0〜14 の範囲外（0 未満または 15 以上）の Intersection に着手が行われる, THEN THE GoBoard SHALL 盤面と Current_Turn を変更せず、着手が無効である旨と無効理由（盤外）を示す結果を返す
3. IF 行番号または列番号のいずれかが整数でない値（非数値、小数、null、未指定を含む）である着手が行われる, THEN THE GoBoard SHALL 盤面と Current_Turn を変更せず、着手が無効である旨と無効理由（不正な座標）を示す結果を返す
4. IF Game_Status が「進行中」以外（勝敗確定、引き分け、未開始を含む）のときに着手が行われる, THEN THE GoBoard SHALL 盤面と Current_Turn を変更せず、着手が無効である旨と無効理由（進行中でない）を示す結果を返す
5. WHEN 盤内かつ空の Intersection に対し Game_Status が「進行中」の状態で着手が行われる, THE GoBoard SHALL 当該 Intersection に Current_Turn の Stone を配置し、着手が有効である旨を示す結果を返す

### Requirement 4: 勝敗判定

**User Story:** プレイヤーとして、5 つ石が並んだら勝ちと判定してほしい。
そうすれば対局の決着がつく。

#### Acceptance Criteria

1. WHEN 着手により Stone が配置され、その Stone を含む縦の Line 上に同じ色の Stone が 5 つ以上連続で並ぶ, THE GoBoard SHALL Game_Status を配置された Stone が黒なら「黒勝ち」、白なら「白勝ち」に設定する
2. WHEN 着手により Stone が配置され、その Stone を含む横の Line 上に同じ色の Stone が 5 つ以上連続で並ぶ, THE GoBoard SHALL Game_Status を配置された Stone が黒なら「黒勝ち」、白なら「白勝ち」に設定する
3. WHEN 着手により Stone が配置され、その Stone を含む右上がり斜めの Line 上に同じ色の Stone が 5 つ以上連続で並ぶ, THE GoBoard SHALL Game_Status を配置された Stone が黒なら「黒勝ち」、白なら「白勝ち」に設定する
4. WHEN 着手により Stone が配置され、その Stone を含む右下がり斜めの Line 上に同じ色の Stone が 5 つ以上連続で並ぶ, THE GoBoard SHALL Game_Status を配置された Stone が黒なら「黒勝ち」、白なら「白勝ち」に設定する
5. WHEN 着手により配置された Stone が複数の Line で同時に 5 つ以上連続を成立させる, THE GoBoard SHALL Game_Status を配置された Stone の色のプレイヤーの勝ち 1 つに設定する
6. WHILE Game_Status が「黒勝ち」または「白勝ち」, THE GoBoard SHALL それ以降の着手を受け付けず、盤面と Current_Turn を変更しない

### Requirement 5: 引き分け判定

**User Story:** プレイヤーとして、盤が埋まっても勝者がいなければ引き分けにしてほしい。
そうすれば決着のつかない対局を正しく終了できる。

#### Acceptance Criteria

1. WHEN 着手の結果、盤上の全 225 個（15×15）の Intersection が Stone で埋まり、かつ Win_Condition が成立していない, THE GoBoard SHALL Game_Status を「引き分け」に設定する
2. WHEN 着手の結果、Game_Status が「引き分け」に設定された, THE GoBoard SHALL 対局が引き分けで終了した旨を示す表示を提示する
3. WHILE Game_Status が「引き分け」, THE GoBoard SHALL それ以降のすべての Intersection への着手要求を拒否し、盤面状態を変更しないまま維持する
4. IF Game_Status が「引き分け」または「勝利」に設定された状態で着手要求を受け取った, THEN THE GoBoard SHALL 当該着手要求を拒否し、対局が既に終了している旨を示す表示を提示する

### Requirement 6: 着手不可の維持

**User Story:** プレイヤーとして、置いた石の位置を後から変えられないようにしてほしい。
そうすれば対局の記録が信頼できる。

#### Acceptance Criteria

1. WHEN 空の Intersection に Stone が配置される, THE GoBoard SHALL その Intersection の色（黒または白）をゲーム終了まで変更せずに保持する
2. IF Stone が置かれている Intersection に対して着手が行われる, THEN THE GoBoard SHALL その着手を拒否し、既存の Stone の色を変更しない
3. IF Stone が置かれている Intersection に対して着手が行われる, THEN THE GoBoard SHALL 手番を進めず、着手前の手番を維持する
4. IF Stone が置かれている Intersection に対して着手が行われる, THEN THE GoBoard SHALL 当該 Intersection が既に埋まっていることを示す着手拒否の通知を返す

### Requirement 7: ビジネスロジックの自動テスト

**User Story:** 開発者として、ロジックの正しさをページ読み込み時に確認したい。
そうすれば回帰を早期に検知できる。

#### Acceptance Criteria

1. WHEN ページの読み込みが完了する, THE Test_Runner SHALL GoBoard のビジネスロジック（盤面状態の管理、勝敗判定、手番制御）に対する全自動テストを、ゲーム UI の初期化前に同期的に実行する
2. WHEN 各テストが実行される, THE Test_Runner SHALL 当該テストのテスト名と成否（成功または失敗）を1件につき1行として console.log に出力する
3. WHEN すべてのテストが完了する, THE Test_Runner SHALL 実行された総テスト数、成功数、失敗数を含むサマリを console.log に出力する
4. IF テストが失敗する, THEN THE Test_Runner SHALL 当該テストのテスト名、期待値、実際の値を含む診断情報を console.log に出力し、後続テストの実行を継続する
5. IF いずれか1件以上のテストが失敗する, THEN THE Test_Runner SHALL 例外を送出せずに全テストの実行を完了させ、ゲームの初期化および UI 表示を通常どおり継続させる
6. WHEN 盤面の初期化に関するテストが実行される, THE Test_Runner SHALL 新しいゲーム開始直後に 225 個すべての Intersection が「空」であり、Current_Turn が Black_Player であり、Game_Status が「進行中」であることを検証する
7. WHEN 交互の着手に関するテストが実行される, THE Test_Runner SHALL 最初の着手で配置される Stone が黒であることを検証する
8. WHEN 交互の着手に関するテストが実行される, THE Test_Runner SHALL 有効な着手が連続するたびに Current_Turn が黒から白、白から黒へ交互に切り替わることを検証する
9. WHEN 着手の妥当性検証に関するテストが実行される, THE Test_Runner SHALL 盤内かつ空の Intersection への着手が受理されることを検証する
10. WHEN 着手の妥当性検証に関するテストが実行される, THE Test_Runner SHALL 既に Stone が置かれている Intersection への着手が拒否されることを検証する
11. WHEN 着手の妥当性検証に関するテストが実行される, THE Test_Runner SHALL 行番号または列番号が 0〜14 の範囲外である Intersection への着手が拒否されることを検証する
12. WHEN 着手の妥当性検証に関するテストが実行される, THE Test_Runner SHALL 行番号または列番号が整数でない不正な座標への着手が拒否されることを検証する
13. WHEN 着手の妥当性検証に関するテストが実行される, THE Test_Runner SHALL Game_Status が「進行中」以外の状態への着手が拒否されることを検証する
14. WHEN 勝敗判定に関するテストが実行される, THE Test_Runner SHALL 縦の Line 上に同じ色の Stone が 5 つ連続したとき、その Stone の色に対応する勝ちが Game_Status に設定されることを検証する
15. WHEN 勝敗判定に関するテストが実行される, THE Test_Runner SHALL 横の Line 上に同じ色の Stone が 5 つ連続したとき、その Stone の色に対応する勝ちが Game_Status に設定されることを検証する
16. WHEN 勝敗判定に関するテストが実行される, THE Test_Runner SHALL 右上がり斜めの Line 上に同じ色の Stone が 5 つ連続したとき、その Stone の色に対応する勝ちが Game_Status に設定されることを検証する
17. WHEN 勝敗判定に関するテストが実行される, THE Test_Runner SHALL 右下がり斜めの Line 上に同じ色の Stone が 5 つ連続したとき、その Stone の色に対応する勝ちが Game_Status に設定されることを検証する
18. WHEN 勝敗判定に関するテストが実行される, THE Test_Runner SHALL いずれの Line 上にも同じ色の Stone が 5 つ連続していない盤面で Win_Condition が成立せず Game_Status が「進行中」のままであることを検証する
19. WHEN 引き分け判定に関するテストが実行される, THE Test_Runner SHALL 全 225 個の Intersection が Stone で埋まり、かつ Win_Condition が成立していない盤面で Game_Status が「引き分け」に設定されることを検証する
20. WHEN 着手不可の維持に関するテストが実行される, THE Test_Runner SHALL 既に Stone が置かれている Intersection へ着手を試みたとき、当該 Intersection の既存の Stone の色と Current_Turn がいずれも変化しないことを検証する
