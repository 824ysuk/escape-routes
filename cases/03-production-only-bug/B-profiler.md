# 事例 3-B: 本番にプロファイラを attach

## 前提 / install

```bash
# py-spy (Python)
pip install py-spy

# async-profiler (JVM) — v3.x 系
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-macos.zip
unzip async-profiler-3.0-macos.zip
# Linux x64: async-profiler-3.0-linux-x64.tar.gz
```

- Linux: `sysctl kernel.yama.ptrace_scope` を確認（0 でないと外部 attach 不可）。一時的に `sudo sysctl -w kernel.yama.ptrace_scope=0`、永続化は `/etc/sysctl.d/10-ptrace.conf` に `kernel.yama.ptrace_scope = 0`
- コンテナ: K8s なら securityContext に `capabilities: { add: ["SYS_PTRACE"] }` を持つ debug pod

## コード

```bash
pgrep -f gunicorn

# py-spy: 30 秒間 sampling して flamegraph SVG を出力
sudo py-spy record -o profile.svg --pid $(pgrep -f gunicorn) --duration 30 --rate 100

# 結果を観察
open profile.svg          # macOS
xdg-open profile.svg      # Linux
explorer.exe profile.svg  # WSL

# async-profiler (JVM) v3.x
./bin/asprof -d 30 -f /tmp/flame.html $(pgrep -f java)
open /tmp/flame.html
```

## 期待出力

- `profile.svg` がブラウザで開き、横軸 = サンプル数（実質 CPU 時間）、縦軸 = call stack の flamegraph
- 横幅が広い関数 = CPU 時間を最も消費
- async-profiler の v3.x では `asprof` コマンド経由（v2.x の `profiler.sh` から変更）

## ハマりポイント

- py-spy は sampling profiler（100ms 未満の短いフェーズは捕捉漏れ）
- container 内では `pgrep` で本物の PID が見えない（namespace 分離）→ host 側で `nsenter -t <PID> -p` 経由
- `Permission denied`: Linux なら `sudo` 必須、または `setcap cap_sys_ptrace=eip /usr/bin/py-spy`
- 観察手法の体系は [Brendan Gregg — Systems Performance](https://www.brendangregg.com/sysperfbook.html) が網羅的
