# 事例 6-A: action-tmate で SSH/web-shell を runner に投入

> **対象範囲**: 自社が運用する CI workflow に限る。public repository では workflow log が誰でも閲覧可能で、tmate session の SSH/web endpoint も log に書き出される。第三者が接続できる構成での起動は禁止 (後述「ハマりポイント」参照)。
> **第三者データ**: 扱わない (自社 runner の内部 state)。ただし他者から接続可能な構成では runner 内 secret / 認証 cache が露出する
> **PII**: 通常扱わないが、runner 内の `env` / cache に request 由来の PII が一時的に残っている可能性がある
> **適法境界**: [GitHub Actions の利用規約](https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features#actions) / 自社の secret 管理規程
> **Disclosure 経路**: [mxschmitt/action-tmate Security Advisories](https://github.com/mxschmitt/action-tmate/security/advisories) / [GitHub Security Lab](https://securitylab.github.com/)

## 前提 / install

- workflow に [mxschmitt/action-tmate](https://github.com/mxschmitt/action-tmate) (v3 以降) を追加できる権限
- GitHub profile に [SSH 公開鍵を登録済](https://github.com/settings/keys) (`limit-access-to-actor: true` で接続を絞るため必須)
- `workflow_dispatch` 経由で再実行する想定で、対象 workflow に `on: workflow_dispatch` を含める
- tmate session 中は runner が占有され続けるため、GitHub-hosted runner の job 上限時間 (default 6 時間) を意識する

## コード

```yaml
on:
  push: {}
  workflow_dispatch:
    inputs:
      debug_enabled:
        description: 'tmate を有効化する'
        required: false
        default: 'false'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: ./scripts/test.sh

      # 失敗時 OR workflow_dispatch で明示有効化したときのみ起動
      - name: Setup tmate session on failure
        if: ${{ failure() || (github.event_name == 'workflow_dispatch' && inputs.debug_enabled == 'true') }}
        uses: mxschmitt/action-tmate@v3
        with:
          limit-access-to-actor: true
          detached: false
```

接続する側のフロー (失敗 workflow の log を開く → "SSH:" / "WEB:" の行をコピー → 自端末で接続):

```bash
# SSH (GitHub profile の鍵で接続)
ssh AbCdEfGhIjKlMnOpQrStUvWxYz@nyc1.tmate.io

# あるいは web 経由 (ブラウザに URL を貼る)
# https://tmate.io/t/AbCdEfGhIjKlMnOpQrStUvWxYz

# 観察したいことの例
env | grep -E '^(CI|GITHUB|RUNNER)_' | sort
cat /etc/os-release
df -h
ls -la "$GITHUB_WORKSPACE"
curl -v https://registry.example.com/healthz

# 観察が終わったら tmate session を抜けて workflow を終了
exit  # あるいは tmate 上で `touch /tmp/keepalive` の解除等を行う
```

## 期待出力

workflow log に 5 秒間隔で連続出力される:

```
SSH: ssh AbCdEfGhIjKlMnOpQrStUvWxYz@nyc1.tmate.io
WEB: https://tmate.io/t/AbCdEfGhIjKlMnOpQrStUvWxYz
SSH: ssh AbCdEfGhIjKlMnOpQrStUvWxYz@nyc1.tmate.io
WEB: https://tmate.io/t/AbCdEfGhIjKlMnOpQrStUvWxYz
```

接続成功時、自端末の terminal に runner 内の prompt (`runner@fv-az...:~/work/<repo>/<repo>$ `) が出る。

## ハマりポイント

### `if:` ガード忘れで成功時にも起動

- ガードを付けないと成功 job でも tmate が起動し、誰も接続しないまま timeout (default 6 時間) まで runner 時間を食う
- GitHub-hosted runner では billing も発生する (job 時間 × runner unit price)
- workflow_dispatch input + `failure()` の OR ガードを default にすると、push 経由でも失敗時のみ起動できる

### 第三者接続のリスク (public repository では特に致命的)

- `limit-access-to-actor: true` を付けない場合、log に流れた SSH/web URL に**任意の第三者が接続可能**になる
- public repository の workflow log は誰でも閲覧可能。endpoint URL も同様に公開される
- 接続された場合、runner 内の `env` / `secrets.*` の展開結果 / checkout 済みコード / cache を読み書きされる
- 検証手順: `limit-access-to-actor: true` 設定後、GitHub profile から SSH 鍵を一時的に外して接続不可になることを確認 (確認後に鍵を戻す)

### `detached: true` で session が切れる

- `detached: true` を指定すると tmate 起動後に workflow が継続する。後続 step が runner を終了させると接続は切れる
- live 観察したい場合は `detached: false` (default) を維持する
- 一方で「後続 step も流したいが、観察のためにバックグラウンドで session を残したい」場合は `detached: true` を選び、最後に `wait-for-tmate-completion-script` 系の step を別途置く

### tmate session の事後トレース

- session 中の操作は tmate 側にも runner 側にも完全な log として残らない (`history` / `script` 等を session 内で別途取らない限り)
- 「何を見て何を判断したか」を Issue / PR に手動メモする
- 観察中に作った一時 file (例: 抽出した env dump) は session 終了で消える。残したいものは artifact upload (事例 6-B) で同 job 内に投げる
