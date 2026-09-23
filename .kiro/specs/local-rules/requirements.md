# Requirements Document

## Introduction

本機能は、既存の GoBoard に「連続3が2箇所同時に成立する着手の禁止（Double_Three 禁止）」という
ローカルルールを追加する。

具体的には、Black_Player の着手を適用した後の盤面全体に、黒の Open_Three（開いた延長可能な端を
持つ、ちょうど 3 個連続した同色の並び）が合計 2 個以上存在する場合、その着手を無効とし石を
配置しない。判定は着手点を通る方向数ではなく、盤面全体に存在する黒の Open_Three の総数で行う。
あわせて、盤面上で成立している Open_Three の位置を記憶する状態を保持し、着手のたびに
この記憶を更新する。黒・白それぞれについて Open_Three が存在するかどうかと個数を
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
- **Open_Three**: 同一 Line 方向上で、同じ色の Stone がちょうど 3 個「連続」して並び（●●●。
  途中に空 Intersection を挟む飛び三 ●●_● は Open_Three に含めない）、かつその並びの両端の
  うち少なくとも一方が空の Intersection であって、その端側へ延長すると相手の Stone または盤端に
  妨げられずに同色 4 個の連続を新たに作り得る状態。両端がいずれも相手の Stone または盤端で
  塞がれ、4 個連続を作り得ない並びは Open_Three ではない。
- **Three_In_A_Row**: 本仕様では Open_Three と同一の状態を指す。すなわち、ちょうど 3 個連続した
  同色の並びのうち、少なくとも一方の端が開いており延長して 4 個連続を作り得るものを指す。
  両端が塞がれた並び、および途中に空を挟む飛び三は含めない。以降、Three_In_A_Row と Open_Three
  は同義として扱う。
- **Win_Condition**: 同一の Line 上に同じ色の Stone が 5 個以上連続し、当該色の勝利が確定する状態。
- **Double_Three**: 1 回の着手を適用した後の盤面全体に、着手したプレイヤーの色の Open_Three が
  合計 2 個以上存在する状態。着手点を通る Line 方向の数ではなく、盤面全体に存在する当該色の
  Open_Three の総数で判定し、複数の Open_Three が共通の Intersection を持つことは要件としない。
- **Three_Registry**: 盤面上で現在成立している Open_Three の位置集合を、黒・白それぞれについて
  記憶する状態。着手が確定するたびに、確定後の盤面に基づいて更新される。Three_Registry は
  連続3の同時2箇所成立の判定と console.log 出力の双方が参照する唯一の情報源であり、
  ログに出力される個数と禁止判定が常に一致する。
- **Test_Runner**: ページ読み込み時にビジネスロジックの自動テストを実行し、結果を出力する仕組み。

## Requirements

### Requirement 1: 黒の連続3が2箇所成立する着手（Double_Three）の禁止

**User Story:** プレイヤーとして、先手（黒）が連続3を同時に2箇所作って一方的に有利にならないようにしてほしい。そうすれば対局の公平性が保たれる。

#### Acceptance Criteria

1. IF Current_Turn が Black_Player であり、盤内の空の Intersection への着手を適用した後の盤面全体に黒の Open_Three（ちょうど 3 個連続し、少なくとも一方の端が開いていて延長により同色 4 個連続を作り得る並び）が合計 2 個以上存在し、かつ当該着手が Win_Condition を成立させない（Double_Three となる）, THEN THE GoBoard SHALL 当該着手を無効とし、Stone を配置せず、盤面状態・Current_Turn・Game_Status を着手前のまま保持し、着手が無効である旨と無効理由（連続3が2箇所同時に成立するため）を示す結果を返す
2. WHEN Current_Turn が Black_Player の着手を適用した後の盤面全体に黒の Open_Three が合計 1 個以下しか存在せず、かつ当該着手が Win_Condition を成立させない, THE GoBoard SHALL 当該着手を有効として受理し、当該 Intersection に黒の Stone を配置する
3. WHEN Current_Turn が White_Player の着手を適用した後の盤面全体に白の Open_Three が合計 2 個以上存在する, THE GoBoard SHALL 連続3の同時2箇所成立の禁止を適用せず、当該着手を有効として受理し、当該 Intersection に白の Stone を配置する
4. WHEN Current_Turn が Black_Player の着手が Double_Three を成立させ、かつ同一の着手がいずれかの Line 上で同色 5 個以上の連続（Win_Condition）を成立させる, THE GoBoard SHALL 連続3の同時2箇所成立の禁止より Win_Condition を優先し、当該着手を有効として受理し、当該 Intersection に黒の Stone を配置し、Game_Status を「黒勝ち」に設定する
5. WHILE 盤面上に黒の Open_Three が既に 1 個存在する, THE GoBoard SHALL Current_Turn が Black_Player の着手が新たにもう 1 個の黒の Open_Three を成立させ盤面全体の黒 Open_Three が合計 2 個となる場合、当該 2 個が互いに交わらず離れた位置にあっても Double_Three として当該着手を無効とする

### Requirement 2: 3 個連続位置の記憶と更新

**User Story:** 開発者として、盤面上で Open_Three が成立している位置を記憶しておきたい。そうすれば連続3の同時成立の判定やデバッグに利用できる。

#### Acceptance Criteria

1. WHEN 新しいゲームが開始される, THE GoBoard SHALL Three_Registry を、黒・白いずれについても Open_Three の位置が 0 個の状態で初期化する
2. WHEN 着手が有効として確定し盤面が更新される, THE GoBoard SHALL 確定後の盤面全体を走査し、黒・白それぞれについて成立している Open_Three（ちょうど 3 個連続し、少なくとも一方の端が開いて延長可能な並び）の位置集合を再計算して Three_Registry を当該結果で置き換える
3. IF 着手が無効として拒否される, THEN THE GoBoard SHALL Three_Registry を着手前の状態のまま変更しない
4. THE Three_Registry SHALL 黒の Open_Three の位置集合と白の Open_Three の位置集合を区別して保持し、同一の Open_Three を重複して保持しない

### Requirement 3: 3 個連続の有無のログ出力

**User Story:** 開発者として、着手のたびに Open_Three の有無と個数をログで確認したい。そうすれば挙動を追跡しやすくなる。

#### Acceptance Criteria

1. WHEN 着手が有効として確定し Three_Registry が更新される, THE GoBoard SHALL Three_Registry が保持する Black_Player の Open_Three が存在するか否か（存在する場合は真、存在しない場合は偽）を、Black_Player に関する出力であると識別できる内容とともに console.log に 1 行出力する
2. WHEN 着手が有効として確定し Three_Registry が更新される, THE GoBoard SHALL Three_Registry が保持する White_Player の Open_Three が存在するか否か（存在する場合は真、存在しない場合は偽）を、White_Player に関する出力であると識別できる内容とともに console.log に 1 行出力する
3. WHEN 着手が有効として確定し Three_Registry が更新され、対象の色の Open_Three が 1 つ以上存在する, THE GoBoard SHALL 当該色について Three_Registry が保持する Open_Three の個数（連続3の同時2箇所成立の禁止の判定に用いる閾値の基準と同一の個数）を含めて console.log に出力する
4. IF 着手が無効として拒否され Three_Registry が更新されない, THEN THE GoBoard SHALL Open_Three の有無に関するログを出力しない

### Requirement 4: 着手拒否の表示

**User Story:** プレイヤーとして、連続3が2箇所同時に成立して着手が拒否されたときに理由を知りたい。そうすれば次の一手を考え直せる。

#### Acceptance Criteria

1. IF Black_Player の着手が Double_Three により無効として拒否される, THEN THE GoBoard SHALL 盤面状態と Current_Turn を拒否前のまま変更せず保持する
2. WHEN Black_Player の着手が Double_Three により無効として拒否される, THE GoBoard SHALL 連続3が2箇所同時に成立するため着手できない旨を、他の着手拒否理由（既に石がある、対局終了済み）と区別できる内容の通知として提示する
3. WHEN 着手が Double_Three 以外の理由（既に石がある、対局終了済み）で拒否される, THE GoBoard SHALL Double_Three による拒否と同一の通知領域を用いて当該の拒否理由を提示する
4. WHEN Double_Three による着手拒否の通知が提示された後に有効な着手が完了する, THE GoBoard SHALL 当該通知領域から連続3の同時2箇所成立に関する通知を消去する

### Requirement 5: 既存ゲームロジックへの影響最小化

**User Story:** 開発者として、ローカルルール追加が既存の実装に与える影響を最小限にしたい。そうすれば回帰リスクを抑えられる。

#### Acceptance Criteria

1. THE GoBoard SHALL 本機能で追加する処理を、連続3の同時成立の判定・着手拒否メッセージの表示・console.log 出力の 3 種に限定し、それ以外の処理を追加しない
2. THE GoBoard SHALL 盤面の初期化・交互の着手・着手の妥当性検証・勝敗判定・引き分け判定・着手不可の維持を、本機能追加の前後で同一の入力に対して同一の結果を返すよう変更せずに維持する
3. THE GoBoard SHALL 盤面状態の管理・勝敗判定・手番制御を担うビジネスロジックを、DOM の参照・生成・操作を行わない純粋な関数として維持する
4. THE GoBoard SHALL HTML + CSS + JavaScript の 1 ファイル構成を維持し、外部ライブラリおよび CDN を一切導入しない
5. IF 本機能を追加した状態でページ読み込み時の自動テストが実行される, THEN THE Test_Runner SHALL 既存 GoBoard の基本ルールに関するテスト（本ローカルルール導入前から存在する 32 件の基本テスト）を引き続きすべて成功させ、当該基本テストの失敗数を増加させない
6. THE GoBoard SHALL 連続3の同時2箇所成立の禁止のルール定義変更に伴い、本ローカルルールに関するテスト（連続3の同時成立の判定・レジストリ・ログのテスト）を新しい全盤面 Open_Three 個数の意味論に合わせて再定義することを許容し、この再定義は前項の基本テストの成功状態には影響を与えない

### Requirement 6: 既存基本ルールの回帰確認

**User Story:** 開発者として、ローカルルール追加後も既存の基本ルールが従来どおり動くことを確認したい。そうすれば安心してリリースできる。

#### Acceptance Criteria

1. WHEN ページの読み込みが完了しテストが実行される, THE Test_Runner SHALL 空の Intersection への着手で Stone が配置されること、着手ごとに Current_Turn が 黒→白→黒 の順で交互に切り替わること、縦・横・右上がり斜め・右下がり斜めのそれぞれで同色 5 連が勝敗判定として認識されることを、個別の検証項目として確認する
2. WHEN 連続3の同時2箇所成立の禁止に関するテストが実行される, THE Test_Runner SHALL Black_Player の着手を適用した後の盤面全体に黒の Open_Three が合計 2 個以上存在する場合に当該着手が無効として拒否され、着手後の盤面と Current_Turn が着手前と同一であることを確認する
3. WHEN 連続3の同時2箇所成立の禁止に関するテストが実行される, THE Test_Runner SHALL Black_Player の着手を適用した後の盤面全体に黒の Open_Three が合計 1 個以下しか存在しない場合に、着手が受理され盤面に Stone が配置されることを確認する
4. WHEN 連続3の同時2箇所成立の禁止に関するテストが実行される, THE Test_Runner SHALL White_Player の着手を適用した後の盤面全体に白の Open_Three が合計 2 個以上存在する場合でも、当該着手が受理され盤面に Stone が配置されることを確認する
5. WHEN 連続3の同時2箇所成立の禁止に関するテストが実行される, THE Test_Runner SHALL 盤面上に黒の Open_Three が既に 1 個存在する状態で、Black_Player の着手が互いに交わらない位置に 2 個目の黒 Open_Three を成立させる場合に、当該着手が Double_Three として拒否されることを確認する
6. IF いずれか 1 件以上のテストが失敗する, THEN THE Test_Runner SHALL 例外を送出せずに全テストの実行を完了させ、ゲームの初期化および UI 表示を通常どおり継続させる
