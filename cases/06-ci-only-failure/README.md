# 事例 6: CI でしか起きない失敗を調査したい

> **対象範囲**: 自分が運用する CI workflow / 自社の CI 環境に限る。public repository の workflow log は誰でも閲覧可能のため、SSH endpoint・debug log への secret 混入・artifact への credential 載せ込みに注意する。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 何が起きるか

CI runner (GitHub Actions / GitLab CI / CircleCI 等) の state は ephemeral で、ジョブ終了とともに失われる。「local では通るが CI でだけ落ちる」状況で、直接ルートが全て詰まる:

- log を読み返しても再現に必要な runner 内部 state (環境変数の実値・キャッシュの内容・network 解決結果) が残っていない
- rerun しても同じ状態で再現するわけではない (キャッシュ更新・並行ジョブ・runner pool の入れ替わり)
- matrix を 1 セルに絞っても、runner 側の事情で挙動が変わると同じ失敗が再現しない
- 「もう一度だけ runner の中を覗ければ解決する」場面で、その手段が手元にない

事例 [3 (本番でしか起きないバグ)](../03-production-only-bug/README.md) は **長期稼働 production プロセスの live 観察**を対象にしており、**ephemeral CI runner** の観察は別軸。runner の生存期間が分〜数十分単位で、観察手段を実行する側もそのウィンドウに収める必要がある。

## 代替手段

| 手段 | 概要 | 侵襲度 |
|---|---|---|
| [A. action-tmate で SSH/web-shell を投入](A-action-tmate.md) | 失敗時のみ runner を SSH/web から live 観察 | 中（runner 占有時間が伸びる） |
| [B. artifact 経由で runner state を持ち出す](B-artifact-state.md) | 失敗時に `/tmp` / `runner.temp` を upload して local 解析 | 低（runner 設定 0、観察は事後） |
| [C. ACTIONS_STEP_DEBUG / RUNNER_DEBUG で詳細 log](C-actions-step-debug.md) | runner 側 debug log を有効化して log の解像度を上げる | 低（log 容量が増えるのみ） |
| [D. nektos/act で CI 環境を local 再現](D-nektos-act.md) | Docker container 上で workflow を local 実行 | 低（local の Docker のみ） |

## 他手段を選ぶ条件

- **A (action-tmate)**: 1 回の失敗で十分情報が取れそう、live で `env` / file / network を触りたい、self-hosted runner で時間制限が緩い
- **B (artifact upload)**: 失敗パターンが複数あり後で diff を取りたい、log / temp file が大量で目視より grep したい、CI 占有時間を伸ばせない
- **C (ACTIONS_STEP_DEBUG)**: runner / action 側で何が評価されたかを知りたい、step の `if:` 条件の真偽が読めない、特定 action 内部のフローを知りたい
- **D (nektos/act)**: 何度も試行錯誤したい、CI を回す cost / 待ち時間を避けたい、network 通信を local proxy で見たい

## 補足

「runner state がもう失われている」を諦め、「失効する前に運び出す (B)」「失効する前に live で覗く (A)」「log の解像度を上げる (C)」「同じ環境を local で作り直す (D)」の 4 方向に分かれる。production 観察 (事例 3) と異なり、観察対象が短時間で消えるため、観察手段の「準備の早さ」が選択を左右する。
