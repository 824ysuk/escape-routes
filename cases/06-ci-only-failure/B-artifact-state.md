# 事例 6-B: artifact 経由で runner state を持ち出す

> **対象範囲**: 自分が運用する CI workflow / 自社の CI 環境。artifact は repository の閲覧権限を持つユーザーが download 可能なため、`secrets.*` を含めない設計にする
> **第三者データ**: 扱わない
> **PII**: 通常扱わないが、runner の `env` / log / cache に request 由来の PII が一時的に残っている可能性がある。upload 対象に `secrets.env` / `.env*` / `~/.docker/config.json` 等を含めない
> **適法境界**: [GitHub Actions の利用規約](https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features#actions) / 自社の secret 管理規程
> **Disclosure 経路**: [actions/upload-artifact issue tracker](https://github.com/actions/upload-artifact/issues) / [GitHub Security Lab](https://securitylab.github.com/)

## 前提 / install

- workflow に [actions/upload-artifact](https://github.com/actions/upload-artifact) v4 を追加できる権限
- artifact の retention は repository の Settings > Actions > "Artifact and log retention" で確認 (default 90 日、organization policy で短縮可)
- free plan の storage quota: 500MB 程度を超えると古い artifact から自動削除される ([Plans and pricing](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions))
- upload 対象の path に secret 含有 file が含まれないかを事前に grep する

## コード

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        id: tests
        run: ./scripts/test.sh

      # 失敗時のみ runner 内部 state を集めて upload
      - name: Collect runner state on failure
        if: ${{ failure() }}
        run: |
          set -u
          STATE_DIR="${RUNNER_TEMP}/failure-state"
          mkdir -p "$STATE_DIR"

          # 1. env (secret は GitHub Actions が `***` に自動マスクするが、念のため一次絞り込み)
          env | grep -vE '^(GITHUB_TOKEN|.*_SECRET|.*_KEY|.*_PASSWORD)=' | sort > "$STATE_DIR/env.txt"

          # 2. system info
          uname -a > "$STATE_DIR/uname.txt"
          cat /etc/os-release > "$STATE_DIR/os-release.txt"
          df -h > "$STATE_DIR/df.txt"
          free -h > "$STATE_DIR/free.txt" || true
          ps auxf > "$STATE_DIR/ps.txt"

          # 3. process が残した log / 一時 file (path はプロジェクト依存)
          cp -r /tmp/*.log "$STATE_DIR/" 2>/dev/null || true
          cp -r "${GITHUB_WORKSPACE}/test-results" "$STATE_DIR/" 2>/dev/null || true

          # 4. upload 直前に secret 残存確認 (allow-list 風の deny grep)
          if grep -rIE '(ghp_|github_pat_|AKIA|xox[bap]-)' "$STATE_DIR/"; then
            echo "::error::secret-like string detected in failure state, aborting upload"
            exit 1
          fi

      - name: Upload runner state on failure
        if: ${{ failure() }}
        uses: actions/upload-artifact@v4
        with:
          name: failure-state-${{ github.run_id }}-${{ github.run_attempt }}
          path: ${{ runner.temp }}/failure-state/
          retention-days: 7
          if-no-files-found: ignore
          # public repository では artifact 内容も public 閲覧可能
```

local で download して解析する側:

```bash
# gh CLI で artifact を取得
gh run download <run-id> --name failure-state-<run-id>-1 --dir ./failure-state
ls failure-state/
# env.txt  uname.txt  os-release.txt  df.txt  free.txt  ps.txt  test-results/

# 想定外の差分を探す例
diff <(cat failure-state/env.txt) <(env | sort) | head -20
```

## 期待出力

upload step の log:

```
Starting artifact upload
Uploading to blob storage
Artifact failure-state-1234567890-1 has been successfully uploaded! Final size is 1843212 bytes.
Artifact download URL: https://github.com/<org>/<repo>/actions/runs/1234567890/artifacts/9876543
```

`gh run download` 後、`failure-state/` 配下に env / system info / log file 一式が並ぶ。

## ハマりポイント

### secret 流出経路

- `env` を無加工で upload すると、`GITHUB_TOKEN` / repository secrets の展開結果 / OIDC token が落ちる可能性がある
- GitHub Actions は log への secret 値を自動 mask するが、**artifact 内の file 内容には mask が適用されない**
- 対策: (a) upload 直前に `grep -rIE` で API key 風文字列を検出して fail させる、(b) `env` を全件落とすのではなく allow-list で whitelist する、(c) [actions/upload-artifact issue #138](https://github.com/actions/upload-artifact/issues/138) を踏まえて secret 含有 file を path から除外する

### artifact の公開範囲

- public repository では artifact も誰でも download 可能
- private repository でも repository read 権限保持者が触れる
- 一度 upload されたものは public archive (Wayback / 第三者 mirror) に残る可能性も想定し、upload 内容を minimize する

### retention / quota

- `retention-days` の上限は repository policy の設定値以下に制限される (例: organization が 30 日制限なら、`retention-days: 90` を指定しても 30 日に丸められる)
- free plan の storage quota (500MB) を超えると古い artifact から自動削除される。debug 用 artifact が大量に貯まると過去分が消える
- `**/*.log` のような大量 match パターンを指定すると upload に数分かかる。事前に `find ... | wc -l` で件数を見ておく

### path に含めるとリスクのある file

- `.env*` / `secrets.env` / `~/.docker/config.json` / `~/.aws/credentials` / `~/.netrc` / OIDC token (`ACTIONS_ID_TOKEN_REQUEST_TOKEN`)
- これらは `.gitignore` 対象であっても runner 上には実体が存在する。明示的に path から除外する
