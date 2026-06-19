# 事例 6-C: ACTIONS_STEP_DEBUG / RUNNER_DEBUG で詳細 log

> **対象範囲**: 自分が運用する CI workflow / 自社の CI 環境。public repository では debug log も public 閲覧可能になり、内部 path / 環境変数名 / step condition の評価過程が露出する
> **第三者データ**: 扱わない
> **PII**: 通常扱わないが、debug log は `env` 展開・action input の評価過程を出力するため、入力経由で PII が log に乗りやすくなる
> **適法境界**: [GitHub Actions の利用規約](https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features#actions) / 自社の secret 管理規程
> **Disclosure 経路**: [GitHub Actions docs feedback](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging) / [GitHub Security Lab](https://securitylab.github.com/)

## 前提 / install

- repository への Settings 書き込み権限 (Secrets / Variables を設定するため)
- [`gh` CLI](https://cli.github.com/) を install 済み (UI からでも設定可能だが gh の方が再現性が高い)
- 対象 workflow が `ubuntu-latest` 等で動くこと (self-hosted runner では runner version が古いと一部 debug 出力が無い)

## コード

```bash
# repository に secret として設定 (variable でも可、後述の注意あり)
gh secret set ACTIONS_STEP_DEBUG --body "true" --repo <org>/<repo>
gh secret set ACTIONS_RUNNER_DEBUG --body "true" --repo <org>/<repo>

# 確認
gh secret list --repo <org>/<repo> | grep ACTIONS_

# workflow を再実行 (push / workflow_dispatch / rerun)
gh workflow run <workflow-file> --repo <org>/<repo>

# log を local に download
gh run view <run-id> --log > run.log

# debug 行のみ抽出
grep -E '^[0-9].*##\[debug\]' run.log | less
```

debug 不要になったら revoke する:

```bash
gh secret delete ACTIONS_STEP_DEBUG --repo <org>/<repo>
gh secret delete ACTIONS_RUNNER_DEBUG --repo <org>/<repo>
```

## 期待出力

workflow log 中、各 step の評価過程・条件・入力解決の経過が `##[debug]` プレフィックスで出力される:

```
##[debug]Evaluating condition for step: 'Run tests'
##[debug]Evaluating: success()
##[debug]Evaluating success:
##[debug]=> true
##[debug]Result: true
##[debug]Starting: Run tests
##[debug]Loading inputs
##[debug]Loading env
##[debug]/usr/bin/bash -e /home/runner/work/_temp/<hash>.sh
```

`if:` 条件が想定外の評価結果になっていたケース、action 入力が `null` に解決されていたケース、cache の key 計算過程の差分等が読み取れる。

## ハマりポイント

### secret と variable の優先順位

- 同名で **secret と variable の両方**を設定した場合、`secrets.*` の方が `vars.*` より優先されるという挙動ではなく、`ACTIONS_STEP_DEBUG` のような GitHub Actions の内部 toggle は **secret 側が優先**される
- variable に `ACTIONS_STEP_DEBUG=false` を入れて「無効化したつもり」になっていても、secret に `true` が残っていれば有効のまま
- 確認: `gh secret list` と `gh variable list` の両方を見る

### 検索性の劣化

- debug log は通常の 10〜100 倍の行数になる
- log search で目的の error message を見つけにくくなる
- 普段 ON にし続けず、調査時のみ ON → 完了後 revoke するフローを default にする

### public repository での露出

- public repository では debug log も誰でも閲覧可能
- 内部 path (`/home/runner/work/_temp/<hash>.sh`)・env 名・action 内部実装の評価過程が公開される
- secret 値そのものは GitHub の自動 mask が効くが、**secret の "存在" や "name"** は隠れない (`##[debug]Evaluating: env.MY_SECRET != ''` のような形で評価が出る)

### 不可逆な情報露出

- 一度 public log として publish された debug 内容は GitHub Actions log retention (default 90 日) 内に削除しても、外部 mirror / 検索 cache に残る可能性
- public repository で debug log を有効化する前に、(a) workflow を fork した状態でまず private repository で試す、(b) public で有効化する場合は最小範囲の 1 push に絞る、を検討する
