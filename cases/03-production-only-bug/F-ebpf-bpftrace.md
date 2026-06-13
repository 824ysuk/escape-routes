# 事例 3-F: eBPF / bpftrace / perf record で kernel-level 観測

> **E との役割分担**: [E-tcpdump-strace.md](E-tcpdump-strace.md) は tcpdump / strace を主軸に bpftrace を補助として紹介している。本ファイルは bpftrace / eBPF / perf を主役に置き、kernel-side の syscall・CPU flame・ディスク I/O・scheduler の長時間観察に特化する。

## 前提 / install

- Linux 4.9+（eBPF stable）、kernel 5.0+ 推奨（BPF ring buffer 等の新機能）
- kernel headers: `sudo apt install linux-headers-$(uname -r)`
- bpftrace: `sudo apt install bpftrace`（Ubuntu 20.04+）
- perf: `sudo apt install linux-tools-common linux-tools-$(uname -r)`
- BCC ツール集（オプション）: `sudo apt install bpfcc-tools`
- container 内で動かす場合: `securityContext.privileged: true` + `bpf` cgroup が必要

```bash
# bpftrace のバージョン確認
bpftrace --version

# kernel の BPF 機能確認
sudo bpftool feature
```

## コード

### CPU flamegraph（ボトルネック特定）

```bash
# 特定プロセスの CPU 時間を flamegraph 化（30 秒サンプリング）
sudo perf record -F 99 -p "$(pgrep -f myapp)" -g -- sleep 30
sudo perf script | stackcollapse-perf.pl | flamegraph.pl > /tmp/flame.svg

# flamegraph ツールが未インストールの場合
git clone --depth 1 https://github.com/brendangregg/FlameGraph.git /tmp/FlameGraph
sudo perf script | /tmp/FlameGraph/stackcollapse-perf.pl | /tmp/FlameGraph/flamegraph.pl > /tmp/flame.svg
```

### syscall 観察（strace 代替、低 overhead）

```bash
# 特定プロセスの syscall を count（strace -c 相当、overhead は圧倒的に低い）
sudo bpftrace -p "$(pgrep -f myapp)" -e '
  tracepoint:syscalls:sys_enter_* { @[probe] = count(); }
  interval:s:10 { print(@); clear(@); }
'

# ファイル open を観察
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_openat /pid == '"$(pgrep -f myapp)"'/ {
    printf("%s %s\n", comm, str(args->filename));
  }
'

# TCP connect を観察（接続先 IP の特定）
sudo bpftrace -e '
  kprobe:tcp_connect {
    $sk = (struct sock *)arg0;
    printf("%-6d %-16s → %s:%d\n",
      pid, comm,
      ntop(2, $sk->__sk_common.skc_daddr),
      $sk->__sk_common.skc_dport >> 8 | ($sk->__sk_common.skc_dport & 0xff) << 8);
  }
'
```

### ディスク latency 分布（I/O スパイク診断）

```bash
# block I/O の latency を histogram で表示
sudo bpftrace -e '
  tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; }
  tracepoint:block:block_rq_complete {
    @usecs = hist((nsecs - @start[args->dev, args->sector]) / 1000);
    delete(@start[args->dev, args->sector]);
  }
  interval:s:5 { print(@usecs); }
'

# BCC out-of-the-box (より高レベル)
sudo biolatency-bpfcc -D    # ディスク別 latency histogram
sudo biosnoop-bpfcc          # 各 I/O のプロセス名 + latency
```

### scheduler latency（CPU wait 時間）

```bash
# プロセスが runqueue に入ってから CPU を得るまでの待ち時間
sudo bpftrace -e '
  tracepoint:sched:sched_wakeup,tracepoint:sched:sched_wakeup_new {
    @qtime[args->pid] = nsecs;
  }
  tracepoint:sched:sched_switch {
    if (@qtime[args->next_pid]) {
      @usecs = hist((nsecs - @qtime[args->next_pid]) / 1000);
      delete(@qtime[args->next_pid]);
    }
  }
  interval:s:5 { print(@usecs); }
'
```

## 期待出力

- `flame.svg`: SVG 形式の flamegraph（ブラウザで開いてクリックで拡大）
- bpftrace syscall count: `@[tracepoint:syscalls:sys_enter_read]: 1234` 形式のカウント表
- ディスク latency histogram: `[0, 1): 5`, `[1, 2): 32`, ... のマイクロ秒分布
- TCP connect: `1234   myapp            → 10.0.2.10:5432` 形式のストリーミング出力
- A/B/D/E の手段より overhead が低く long-run 観察が可能（perf record の CPU 使用率は通常 1% 未満）

## ハマりポイント

### container 内での BPF 実行

- `bpf()` syscall が seccomp で blocked される場合がある → `--privileged` または `securityContext.allowPrivilegeEscalation: true` + `capabilities.add: ["SYS_BPF", "PERFMON"]` が必要
- Kubernetes では ephemeral container を使う方が安全: `kubectl debug -it <pod> --image=ubuntu:22.04 --target=<container>`
- BPF program は kernel-side で verifier を通過する必要がある。複雑なロジックは BCC / libbpf 経由で書く（bpftrace の one-liner ではカバーできないケースもある）

### kernel version の差異

- probe 名（tracepoint / kprobe の引数名）が kernel version によって変わる
- `bpftrace -l 'tracepoint:syscalls:sys_enter_openat'` で利用可能な probe を先に確認する
- CO-RE（Compile Once – Run Everywhere）対応は libbpf / bpftrace 0.16+ 推奨

### perf record の symbol 解決

- JIT コンパイル言語（Node.js / JVM / Python）は perf の symbol 解決に追加設定が必要:
  - Node.js: `--perf-basic-prof` フラグで `/tmp/perf-<pid>.map` を生成
  - JVM: `-XX:+PreserveFramePointer` + `perf-map-agent`
  - Python: `py-spy` の方が手軽（[B-profiler.md](B-profiler.md) 参照）

### セキュリティ・機密取扱い

- bpftrace は任意の kernel メモリを読める → production での使用は監査ログを残す
- `/tmp` への書き出しは world-readable: `install -d -m 700 /var/tmp/perf-$USER` で専用 dir を作成
- flamegraph SVG にはシンボル名（関数名・引数）が含まれる。Slack / Issue に貼る前に機密シンボルが含まれていないか確認する

### 参考

- [Brendan Gregg - BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html)
- [iovisor/bcc tools](https://github.com/iovisor/bcc/tree/master/tools)
- [bpftrace reference guide](https://github.com/bpftrace/bpftrace/blob/master/docs/reference_guide.md)
- [Brendan Gregg - FlameGraph](https://github.com/brendangregg/FlameGraph)
- [Linux perf wiki](https://perf.wiki.kernel.org/index.php/Main_Page)
