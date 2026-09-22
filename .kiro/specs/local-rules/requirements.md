# Requirements Document

## Introduction

本機能は、既存の GoBoard に「三三の禁止（Double_Three 禁止）」というローカルルールを追加する。

具体的には、Black_Player の着手が、その一手によって異なる 2 つ以上の方向に同時に
Open_Three（両端が空いた、ちょうど 3 個連続した同色の並び）を成立させる場合、その着手を無効とし
石を配置しない。あわせて、盤面上で 3 個連続が成立している位置を記憶する状態を保持し、
着手のたびにこの記憶を更新する。黒・白それぞれについて 3 個連続の位置が存在するかどうかを
console.log に出力する。

本追加は、既存のゲームロジックへの影響範囲を最小化する形で行う。追加対象は
ルール判定・着手拒否メッセージの表示・console.log 出力に限り、既存の
盤面初期化・交互着手・勝敗判定・引き分け判定などの振る舞いは変更しない。

実装は引き続き HTML + CSS + JavaScript を 1 ファイルで完結させ、外部ライブラリおよび
CDN は使用しない。ビジネスロジックは DOM に依存しない純粋な関数として切り出し、
ページ読み込み時に自動テストを実行して結果を console.log に出力する。

## Glossary

- **GoBoard**: 本ゲーム全体、およびゲーム状態を管理するビジネスロジックのシステム名。
- **Board**: 15×15 の交点からなる格子盤。各交点は「空」「黒」「白」のいずれかの状態を持つ。
- **Intersection**: 盤面を構成する縦線と横線が交わる点。
  各 Intersection は行番号 row（0〜14）と列番号 col（0〜14）の組で一意に識別される。
- **Stone**: 交点に置かれる石。色は黒または白。
- **Black_Player**: 先手のプレイヤー。黒石を使用する。
- **White_Player**: 後手のプレイヤー。白石を使用する。
- **Current_Turn**: 現在手番を持つプレイヤーを表す状態（黒または白）。
- **Line**: 縦・横・右上がり斜め・右下がり斜めのいずれかの直線方向。
- **Game_Status**: ゲームの進行状態。「進行中」「黒勝ち」「白勝ち」「引き分け」のいずれか。
- **Three_In_A_Row**: 同一の Line 上に同じ色の Stone がちょうど 3 個連続で並んだ状態。
  両端が空いているかどうかは問わない。
- **Open_Three**: 同一 Line 方向上で、対象の Stone を含む同色ちょうど 3 個が、連続または内部に
  1 個の空 Intersection を 1 か所だけ挟んで並び、かつその並びの両端側を延長して、相手の Stone
  または盤端に妨げられずに同色 4 個の連続（Open_Four に発展し得る形）を新たに作り得る状態。
  両端側のいずれもが相手の Stone または盤端で塞がれ 4 個連続を作り得ない場合は Open_Three では
  ない。
- **Win_Condition**: 同一の Line 上に同じ色の Stone が 5 個以上連続し、当該色の勝利が確定する状態。
- **Double_Three**: 1 回の着手により、着手した Stone を含む Open_Three が異なる 2 つ以上の
  Line 方向で同時に成立する状態。
- **Three_Registry**: 盤面上で現在成立している Three_In_A_Row の位置を、黒・白それぞれについて
  記憶する状態。着手が確定するたびに、確定後の盤面に基づいて更新される。
- **Test_Runner**: ページ読み込み時にビジネスロジックの自動テストを実行し、結果を出力する仕組み。

## Requirements

### Requirement 1: 黒の三三（Double_Three）着手の禁止

**User Story:** プレイヤーとして、先手（黒）が三三で一方的に有利にならないようにしてほしい。そうすれば対局の公平性が保たれる。

#### Acceptance Criteria

1. IF Current_Turn が Black_Player かつ、盤内の空の Intersection への着手が、着手した Stone を含む Open_Three（同一 Line 方向上で当該 Stone を含む同色ちょうど 3 個が、連続または内部に 1 個の空 Intersection を 1 か所だけ挟んで並び、かつその並びの両端側を延長して相手の Stone または盤端に妨げられずに同色 4 個の連続を新たに作り得る状態）を、縦・横・右上がり斜め・右下がり斜めの 4 方向のうち相異なる 2 方向以上で同時に成立させる（Double_Three となる）, THEN THE GoBoard SHALL 当該着手を無効とし、Stone を配置せず、盤面状態・Current_Turn・Game_Status を着手前のまま保持し、着手が無効である旨と無効理由（三三禁止）を示す結果を返す
2. WHEN Current_Turn が Black_Player の着手が、着手した Stone を含む Open_Three を、4 方向のうちちょうど 1 方向でのみ成立させ、かつ Win_Condition を成立させない, THE GoBoard SHALL 当該着手を有効として受理し、当該 Intersection に黒の Stone を配置する
3. WHEN Current_Turn が White_Player の着手が、4 方向のうち相異なる 2 方向以上で同時に Open_Three を成立させる, THE GoBoard SHALL 三三禁止を適用せず、当該着手を有効として受理し、当該 Intersection に白の Stone を配置する
4. WHEN Current_Turn が Black_Player の着手が Double_Three を成立させ、かつ同一の着手がいずれかの Line 上で同色 5 個以上の連続（Win_Condition）を成立させる, THE GoBoard SHALL 三三禁止より Win_Condition を優先し、当該着手を有効として受理し、当該 Intersection に黒の Stone を配置し、Game_Status を「黒勝ち」に設定する

### Requirement 2: 3 個連続位置の記憶と更新

**User Story:** 開発者として、盤面上で 3 個連続が成立している位置を記憶しておきたい。そうすれば三三判定やデバッグに利用できる。

#### Acceptance Criteria

1. WHEN 新しいゲームが開始される, THE GoBoard SHALL Three_Registry を、黒・白いずれについても Three_In_A_Row の位置が 0 個の状態で初期化する
2. WHEN 着手が有効として確定し盤面が更新される, THE GoBoard SHALL 確定後の盤面全体を走査し、黒・白それぞれについて成立している Three_In_A_Row の位置集合を再計算して Three_Registry を当該結果で置き換える
3. IF 着手が無効として拒否される, THEN THE GoBoard SHALL Three_Registry を着手前の状態のまま変更しない
4. THE Three_Registry SHALL 黒の Three_In_A_Row の位置集合と白の Three_In_A_Row の位置集合を区別して保持し、同一の Three_In_A_Row を重複して保持しない

### Requirement 3: 3 個連続の有無のログ出力

**User Story:** 開発者として、着手のたびに 3 個連続の有無をログで確認したい。そうすれば挙動を追跡しやすくなる。

#### Acceptance Criteria

1. WHEN 着手が有効として確定し Three_Registry が更新される, THE GoBoard SHALL Black_Player の Stone による Three_In_A_Row が存在するか否か（存在する場合は真、存在しない場合は偽）を、Black_Player に関する出力であると識別できる内容とともに console.log に 1 行出力する
2. WHEN 着手が有効として確定し Three_Registry が更新される, THE GoBoard SHALL White_Player の Stone による Three_In_A_Row が存在するか否か（存在する場合は真、存在しない場合は偽）を、White_Player に関する出力であると識別できる内容とともに console.log に 1 行出力する
3. WHEN 着手が有効として確定し Three_Registry が更新され、対象の色の Three_In_A_Row が 1 つ以上存在する, THE GoBoard SHALL 当該色について存在する Three_In_A_Row の個数を含めて console.log に出力する
4. IF 着手が無効として拒否され Three_Registry が更新されない, THEN THE GoBoard SHALL Three_In_A_Row の有無に関するログを出力しない

### Requirement 4: 着手拒否の表示

**User Story:** プレイヤーとして、三三で着手が拒否されたときに理由を知りたい。そうすれば次の一手を考え直せる。

#### Acceptance Criteria

1. IF Black_Player の着手が Double_Three により無効として拒否される, THEN THE GoBoard SHALL 盤面状態と Current_Turn を拒否前のまま変更せず保持する
2. WHEN Black_Player の着手が Double_Three により無効として拒否される, THE GoBoard SHALL 三三禁止により着手できない旨を、他の着手拒否理由（既に石がある、対局終了済み）と区別できる内容の通知として提示する
3. WHEN 着手が Double_Three 以外の理由（既に石がある、対局終了済み）で拒否される, THE GoBoard SHALL Double_Three による拒否と同一の通知領域を用いて当該の拒否理由を提示する
4. WHEN Double_Three による着手拒否の通知が提示された後に有効な着手が完了する, THE GoBoard SHALL 当該通知領域から三三禁止の通知を消去する

### Requirement 5: 既存ゲームロジックへの影響最小化

**User Story:** 開発者として、ローカルルール追加が既存の実装に与える影響を最小限にしたい。そうすれば回帰リスクを抑えられる。

#### Acceptance Criteria

1. THE GoBoard SHALL 本機能で追加する処理を、三三判定・着手拒否メッセージの表示・console.log 出力の 3 種に限定し、それ以外の処理を追加しない
2. THE GoBoard SHALL 盤面の初期化・交互の着手・着手の妥当性検証・勝敗判定・引き分け判定・着手不可の維持を、本機能追加の前後で同一の入力に対して同一の結果を返すよう変更せずに維持する
3. THE GoBoard SHALL 盤面状態の管理・勝敗判定・手番制御を担うビジネスロジックを、DOM の参照・生成・操作を行わない純粋な関数として維持する
4. THE GoBoard SHALL HTML + CSS + JavaScript の 1 ファイル構成を維持し、外部ライブラリおよび CDN を一切導入しない
5. IF 本機能を追加した状態でページ読み込み時の自動テスト（Requirement 6 で定義された全テスト）が実行される, THEN THE Test_Runner SHALL 本機能追加前に成功していたすべてのテストを引き続き成功させ、失敗数を増加させない

### Requirement 6: 既存基本ルールの回帰確認

**User Story:** 開発者として、ローカルルール追加後も既存の基本ルールが従来どおり動くことを確認したい。そうすれば安心してリリースできる。

#### Acceptance Criteria

1. WHEN ページの読み込みが完了しテストが実行される, THE Test_Runner SHALL 空の Intersection への着手で Stone が配置されること、着手ごとに Current_Turn が 黒→白→黒 の順で交互に切り替わること、縦・横・右上がり斜め・右下がり斜めのそれぞれで同色 5 連が勝敗判定として認識されることを、個別の検証項目として確認する
2. WHEN 三三禁止に関するテストが実行される, THE Test_Runner SHALL Black_Player の Double_Three となる着手が無効として拒否され、着手後の盤面と Current_Turn が着手前と同一であることを確認する
3. WHEN 三三禁止に関するテストが実行される, THE Test_Runner SHALL Black_Player の着手がちょうど 1 方向でのみ Open_Three を成立させる場合に、着手が受理され盤面に Stone が配置されることを確認する
4. WHEN 三三禁止に関するテストが実行される, THE Test_Runner SHALL White_Player の Double_Three となる着手が受理され盤面に Stone が配置されることを確認する
5. IF いずれか 1 件以上のテストが失敗する, THEN THE Test_Runner SHALL 例外を送出せずに全テストの実行を完了させ、ゲームの初期化および UI 表示を通常どおり継続させる
