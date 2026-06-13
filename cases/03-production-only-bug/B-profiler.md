# 事例 3-B: 本番にプロファイラを attach

## 前提 / install

```bash
# py-spy (Python)
pip install py-spy

# async-profiler (JVM) — 最新版を使う
# https://github.com/async-profiler/async-profiler/releases から arch / libc に合わせて取得
# Linux glibc x86_64:  async-profiler-<latest>-linux-x64.tar.gz
# Linux musl (Alpine): async-profiler-<latest>-linux-musl-x64.tar.gz
# macOS arm64+x86_64:  async-profiler-<latest>-macos.zip
VERSION=3.0   # 執筆時点。最新は releases を参照
wget "https://github.com/async-profiler/async-profiler/releases/download/v${VERSION}/async-profiler-${VERSION}-macos.zip"
unzip "async-profiler-${VERSION}-macos.zip"
./bin/asprof -v   # version 確認
```

- Linux: `sysctl kernel.yama.ptrace_scope` を確認（0 でないと外部 attach 不可）。**`0` 永続化は禁止** — 同 uid プロセスを互いに ptrace 可能にする最も緩い設定で、低権限 RCE 後に同 uid で動く他プロセスの credential / JWT 等を抜かれる経路になる（[Yama LSM docs](https://www.kernel.org/doc/Documentation/admin-guide/LSM/Yama.rst)）。代わりに attach の前後で **一時切替 + 復元** をワンセットで実行する:
  ```bash
  ORIG_SCOPE=$(cat /proc/sys/kernel/yama/ptrace_scope)
  sudo sysctl -w kernel.yama.ptrace_scope=1   # default。同 uid のみ
  # ... attach 中 ...
  sudo sysctl -w kernel.yama.ptrace_scope="$ORIG_SCOPE"   # 終了後に必ず復元
  ```
- コンテナ: K8s なら debug pod に `securityContext.capabilities.add: ["SYS_PTRACE"]` + `shareProcessNamespace: true` を持たせて隔離する（[Kubernetes securityContext docs](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)）。本番 Pod 全体の `ptrace_scope` を弄らない

## コード

```bash
# gunicorn は master + N workers 構成。pgrep は複数 PID を返すため
# worker を 1 つ選ぶか、--subprocesses で master 経由に集約する
pgrep -f 'gunicorn.*worker'

# py-spy: 単一 worker を 30 秒 sampling
WORKER=$(pgrep -f 'gunicorn.*worker' | head -1)
sudo py-spy record -o profile.svg --pid "$WORKER" --duration 30 --rate 100

# 全 worker を集約したい場合は master 経由 (--subprocesses)
MASTER=$(pgrep -of gunicorn)   # 最古プロセス = master
sudo py-spy record --subprocesses -o profile.svg --pid "$MASTER" --duration 30

# 結果を観察
open profile.svg          # macOS
xdg-open profile.svg      # Linux
explorer.exe profile.svg  # WSL

# async-profiler (JVM) — JVM PID を明示
JVM=$(pgrep -f java | head -1)
./bin/asprof -d 30 -f /tmp/flame.html "$JVM"
open /tmp/flame.html
```

## 期待出力

- `profile.svg` がブラウザで開き、横軸 = サンプル数（実質 CPU 時間）、縦軸 = call stack の flamegraph
- 横幅が広い関数 = CPU 時間を最も消費
- async-profiler の v3.x では `asprof` コマンド経由（v2.x の `profiler.sh` から変更）

## ハマりポイント

- py-spy は sampling profiler（100ms 未満の短いフェーズは捕捉漏れ）
- `pgrep -f gunicorn` は master + N workers の複数 PID を返す。stdout を `--pid` に渡すと py-spy が error する。`pgrep -f 'gunicorn.*worker' | head -1` で worker を 1 つ選ぶか、`pgrep -of gunicorn` で master を取り `--subprocesses` で全 worker 集約（[py-spy README](https://github.com/benfred/py-spy#how-do-i-profile-a-process-with-subprocesses)）
- async-profiler のバージョン: 上のコードは `VERSION` 変数で切り替える形にしているが、release tag が yanked される / 後続版で bugfix が入る前提で、必ず最新版を [releases](https://github.com/async-profiler/async-profiler/releases) で確認する。Alpine 系では `linux-musl-x64.tar.gz` を選ぶ（glibc 用は libc 不整合で動かない）
- container 内では `pgrep` で本物の PID が見えない（namespace 分離）→ host 側で `nsenter -t <PID> -p` 経由
- `Permission denied`: Linux なら `sudo` 必須、または `setcap cap_sys_ptrace=eip /usr/bin/py-spy`
- `ptrace_scope` は必ず attach 終了後に元値へ戻す（前提セクションのスクリプト参照）。永続化禁止
- 観察手法の体系は [Brendan Gregg — Systems Performance](https://www.brendangregg.com/sysperfbook.html) が網羅的
