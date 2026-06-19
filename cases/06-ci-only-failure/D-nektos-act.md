# 事例 6-D: nektos/act で CI 環境を local 再現

> **対象範囲**: 自分の local 開発環境のみ。act は GitHub の API key / secret を local の `.env` / `.secrets` から読むため、これらの file の管理に注意する
> **第三者データ**: 扱わない
> **PII**: 扱わない (実 runner 上の state を持ち出すわけではなく、local の Docker 上で workflow を実行する)
> **適法境界**: nektos/act は MIT License。GitHub Actions の self-hosted runner 規約は GitHub-hosted runner の代替として local 実行を制限しない
> **Disclosure 経路**: [nektos/act Security Advisories](https://github.com/nektos/act/security/advisories) / [catthehacker image 提供元 issue tracker](https://github.com/catthehacker/docker_images/issues)

## 前提 / install

- Docker Engine / Docker Desktop が動作する環境
- Apple Silicon (M1/M2/M3) では `--container-architecture linux/amd64` を必ず付ける (後述)
- [act](https://github.com/nektos/act) install:

```bash
# macOS (Homebrew)
brew install act

# Linux (bash one-liner、上流推奨)
curl -fsSL https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# version 確認
act --version
```

## コード

```bash
# 1. workflow file 一覧と job 名を確認
act -l

# 2. 特定 job のみ実行 (default の micro image はツール不足のため medium image を明示)
act -j build -P ubuntu-latest=catthehacker/ubuntu:act-latest

# 3. Apple Silicon は amd64 強制
act -j build \
  --container-architecture linux/amd64 \
  -P ubuntu-latest=catthehacker/ubuntu:act-latest

# 4. secret / env を渡す (.secrets file は .gitignore に追加しておく)
cat > .secrets <<'EOF'
GITHUB_TOKEN=ghp_xxxxxxxxxxxx
NPM_TOKEN=npm_xxxxxxxxxxxx
EOF
chmod 600 .secrets

act -j build \
  -P ubuntu-latest=catthehacker/ubuntu:act-latest \
  --secret-file .secrets

# 5. 1 step だけ実行を試す (dry-run で plan 確認)
act -j build --dryrun

# 6. 永続化したい設定は .actrc に
cat > .actrc <<'EOF'
-P ubuntu-latest=catthehacker/ubuntu:act-latest
--container-architecture linux/amd64
EOF
```

## 期待出力

```
[CI/build] 🚀  Start image=catthehacker/ubuntu:act-latest
[CI/build]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform= username= forcePull=false
[CI/build]   🐳  docker create image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[] network=""
[CI/build]   🐳  docker run image=catthehacker/ubuntu:act-latest platform=linux/amd64 entrypoint=["tail" "-f" "/dev/null"] cmd=[]
[CI/build] ⭐ Run Main actions/checkout@v4
[CI/build]   ✅  Success - Main actions/checkout@v4
[CI/build] ⭐ Run Main Run tests
[CI/build]   ❌  Failure - Main Run tests
[CI/build] exitcode '1': failure
```

失敗時は exit code が伝搬する。`docker ps -a` で container が残っており、`docker exec -it <id> bash` で中に入って state を観察可能。

## ハマりポイント

### default image (micro) のツール不足

- act の default image は `node:16-buster-slim` 派生で、Python / Docker / 多くの GitHub-hosted runner 同梱ツール (jq / gh / awscli 等) を欠く
- 本番 workflow が「local の act だけ落ちる」逆転が起きる
- 対策: `-P ubuntu-latest=catthehacker/ubuntu:act-latest` で medium image (約 1.5GB) を明示する
- さらに本番に近づけたい場合は [catthehacker/docker_images](https://github.com/catthehacker/docker_images) の full image (約 18GB) を選ぶが、disk 消費が大きい

### Apple Silicon での platform mismatch

- M1/M2/M3 (arm64) では `--container-architecture linux/amd64` を付けないと「runner platform mismatch」で起動失敗する
- `.actrc` に常駐させると毎回付け忘れがなくなる
- Rosetta 経由のエミュレーションになるため、x86 ネイティブ Docker と比較して 30〜50% 遅い

### 再現不可能な領域

- Docker container ベースのため、systemd を要する step (service の `systemctl start` 等) は再現不可
- cgroup v2 操作・カーネル module / sysctl を触る step は failed か silent miss
- GitHub-hosted runner と異なる kernel / libc バージョンの差で挙動が変わるケース (例: glibc 依存 ldconfig 等)
- `actions/cache` の動作は [tonistiigi/act-cache](https://github.com/tonistiigi/act-cache) 等の手当てが要る

### secret / token の取り扱い

- `--secret-file .secrets` で渡したものは container 内 env として展開される
- `.secrets` 自体を repository にコミットしない (`.gitignore` 追加 + `gitleaks` で commit 前検出)
- `GITHUB_TOKEN` を渡す場合は本番権限を持つ token ではなく、scope を絞った PAT を使う

### GitHub-hosted runner との根本的差分

- act は self-hosted runner 互換ではない (`runner.os` / `runner.tool_cache` が異なる)
- `${{ runner.temp }}` が GitHub-hosted runner の path と一致しない
- 「local の act では通るが GitHub-hosted で落ちる」逆方向の不一致もあり得るため、act の合否は「本番 CI に近い再現」であり「本番 CI そのもの」ではないと位置付ける
