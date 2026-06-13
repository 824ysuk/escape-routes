# 事例 3-B: 本番にプロファイラを attach

> **対象範囲**: 自社が運用するプロダクション環境への attach に限る。production への動的観測は SLO に影響する可能性があるため、変更管理・観測時間枠の合意を取った上で実施する。

## 前提 / install

```bash
# py-spy (Python)
pip install py-spy

# async-profiler (JVM) — v3.x 系 (Linux production を default に)
ARCH=$(uname -m)        # x86_64 → x64, aarch64 → arm64
wget https://github.com/async-profiler/async-profiler/releases/latest/download/async-profiler-linux-x64.tar.gz
tar xf async-profiler-linux-x64.tar.gz
# macOS / Windows は releases page から該当 archive を選ぶ
```

### Linux kernel の前提

production の security posture を破壊しない順序で attach する。

| sysctl | 推奨値 | 意味 |
|---|---|---|
| `kernel.yama.ptrace_scope` | **`1` (restricted ptrace, default on Ubuntu/Debian)** | 親プロセス・同一 UID + `prctl(PR_SET_PTRACER, ...)` 許可のみ attach 可 |
| `kernel.perf_event_paranoid` | 観測中 `1` (可能なら `-1`) | async-profiler が perf_event_open で kernel sample / PMU カウンタを読むのに必要 |
| `kernel.kptr_restrict` | `0` | kernel symbol を解決 (flamegraph に kernel frame を含めるなら) |
| `kernel.perf_event_max_stack` | `1024` | 深い stack を取りこぼさない |

**`ptrace_scope=0` への変更は推奨しない**。`0` にすると同一 UID のあらゆるプロセスが互いを ptrace 可能になり、攻撃者が memory から credential を抽出する経路 (CVE-2019-13272 の前提条件等) を恒久的に開く。

**`ptrace_scope=1` のまま attach する 3 つの経路**:

1. **同一 UID で実行**: `sudo -u <app_user> py-spy record ...` または app プロセスと同じ UID から起動
2. **container 内で `--cap-add=SYS_PTRACE`**: Docker / K8s で SYS_PTRACE capability を attach 元プロセスに付与
3. **K8s ephemeral debug container** (Kubernetes 1.25+): `kubectl debug -it <pod> --image=python:3.12 --target=<container> --profile=sysadmin` (`--profile=sysadmin` が自動で SYS_PTRACE を付与する)

容易に対応できない場合に限り、観測時間中だけ `0` に下げ完了後 `1` に戻す:
```bash
ORIG=$(sysctl -n kernel.yama.ptrace_scope)
sudo sysctl -w kernel.yama.ptrace_scope=0
# ... attach 観測 ...
sudo sysctl -w kernel.yama.ptrace_scope="$ORIG"
```

### Container での PID 解決

container 内 PID と host PID は別 namespace。host から見える PID を `/proc/<host_pid>/status` の `NSpid:` 行で照合する:

```bash
# host 側で対象 container 内の app PID を取得
HOST_PID=$(docker inspect --format '{{.State.Pid}}' my-app-container)
# または K8s で
HOST_PID=$(kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[0].containerID}' | xargs -I{} crictl inspect {} | jq '.info.pid')
sudo grep NSpid /proc/$HOST_PID/status   # NSpid: <host>  <container>
```

## コード

```bash
APP_PID=$(pgrep -f gunicorn)

# py-spy: production 推奨フラグ
sudo py-spy record \
  -o profile.svg \
  --pid "$APP_PID" \
  --duration 30 \
  --rate 250 \
  --idle \
  --native \
  --subprocesses
#   --rate 250  : 100Hz では短い request の sampling 誤差が大きい
#   --idle      : GIL を解放した sleeping thread を 0% に消さない
#   --native    : C 拡張 (libssl, numpy) の時間を独立に可視化
#   --subprocesses : gunicorn / uwsgi の worker をまとめて取る

# 結果を観察
open profile.svg          # macOS
xdg-open profile.svg      # Linux
explorer.exe profile.svg  # WSL

# async-profiler (JVM) v3.x — 観測対象別の event タイプ
./bin/asprof -e cpu   -d 30 -f cpu.html  "$APP_PID"   # CPU bound
./bin/asprof -e alloc -d 60 -f alloc.html "$APP_PID"  # GC pressure (TLAB sampling)
./bin/asprof -e lock  -d 30 -f lock.html "$APP_PID"   # 競合
./bin/asprof -e wall  -t -d 30 -f wall.html "$APP_PID"  # off-CPU (wall-clock + per-thread)

# perf_event が取れない環境では fallback
./bin/asprof -e itimer -d 30 -f cpu.html "$APP_PID"   # user-only fallback
./bin/asprof -e ctimer -d 30 -f cpu.html "$APP_PID"   # container / cgroup v2 向け
```

## 期待出力

- `profile.svg` がブラウザで開き、横軸 = サンプル数（実質 CPU 時間）、縦軸 = call stack の flamegraph
- 横幅が広い関数 = CPU 時間を最も消費
- async-profiler v3.x は `asprof` コマンド経由（v2.x の `profiler.sh` から変更）

## ハマりポイント

### profiler バイアス

- py-spy は sampling profiler。`--idle` 無しだと C 拡張で sleeping している thread が CPU 0% に見える
- `--native` 無しだと `libssl` / `numpy` C コード内の時間が parent Python frame に集約されて誤分析
- 100 Hz は短い request で誤差大、250 Hz 以上推奨

### async-profiler / perf_event

- container 内では host の `/proc/sys` 値が継承され、container 内 sysctl では変えられない
- async-profiler は SIGPROF を奪うため、他のプロファイラ (例: gunicorn の `--statsd-host`) と競合する場合あり
- JDK 17+ では `jcmd <PID> JFR.start` でも async-profiler 相当のことができる

### 権限

- `Permission denied`: 上記「3 つの経路」を選ぶ
- file capability `setcap cap_sys_ptrace=eip /usr/bin/py-spy` は dynamic linker が secure-execution mode に入り `LD_LIBRARY_PATH` / `LD_PRELOAD` を無視するため、カスタム lib に依存する binary では使えない
- package update で binary が置換されると cap も消える → container では `securityContext.capabilities.add: ["SYS_PTRACE"]` を Pod spec に書く方が再現性が高い

### 参考

- 観察手法の体系: [Brendan Gregg — Systems Performance (2nd ed)](https://www.brendangregg.com/sysperfbook.html)
- [Linux perf-security docs](https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html)
- [async-profiler README - profiler options](https://github.com/async-profiler/async-profiler#profiler-options)
- [Linux Yama LSM (ptrace_scope) docs](https://www.kernel.org/doc/html/latest/admin-guide/LSM/Yama.html)
