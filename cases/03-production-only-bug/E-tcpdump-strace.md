# 事例 3-E: tcpdump / strace / bpftrace

> **対象範囲**: 自社が運用するプロダクション環境への観測に限る。production への動的観測は SLO に影響しうるため、観測手段は overhead を理解した上で選ぶ。

## 観測手段の選択フロー

production で「アプリログに何も出ない」状況で OS / network 境界を疑うとき、以下の順序で選ぶ:

| 観測対象 | 第一選択 (低 overhead) | fallback |
|---|---|---|
| syscall 全般 | `perf trace` / `bpftrace` | `strace -e` で絞る |
| ファイル open | `bpftrace -e 'tracepoint:syscalls:sys_enter_openat ...'` | `strace -e trace=openat` |
| network connect | `tcpconnect.bt` / `tcpdump` | `strace -e trace=connect` |
| 長時間観測 | `tcpdump -C -W -G` 循環 + display filter で post-analysis | - |
| HTTPS 内部観察 | [mitmproxy (事例 5-A)](../05-invisible-traffic/A-mitmproxy.md) | SSLKEYLOGFILE で Wireshark 復号 |

**strace は最終手段**。ptrace ベースで syscall ごとに 2 回 context switch を入れるため、実測で 100x slowdown が起きうる ([brendangregg.com: strace Wow Much Syscall](https://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html))。production の SLO に直撃する。

## 前提 / install

```bash
brew install tcpdump            # macOS
sudo apt install tcpdump        # Debian / Ubuntu
brew install --cask wireshark   # GUI + tshark
sudo apt install strace         # Linux 専用
sudo apt install bpftrace       # 推奨 (Linux 5.0+ で動作)
sudo apt install linux-tools-common linux-tools-$(uname -r)  # perf
# macOS は dtruss / dtrace (プリインストール)
```

- sudo 権限必須 (または `setcap` で per-binary に capability 付与、下記参照)
- コンテナ: `kubectl debug` で ephemeral container を attach、または sidecar 追加

## コード

### 第一選択: perf trace / bpftrace (低 overhead)

```bash
# perf trace: syscall を観察 (strace 同等の情報量、kernel side で動くため overhead が小さい)
sudo perf trace -p "$(pgrep -f myapp)"

# 短時間で flamegraph も取れる (CPU bound 解析と兼用)
sudo perf record -F 99 -ag -- sleep 30
sudo perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg

# bpftrace で特定 syscall を観察
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat /pid == '"$(pgrep -f myapp)"'/ {
  printf("%s %s\n", comm, str(args->filename));
}'

# bcc-tools の出来合い (apt install bpfcc-tools)
sudo opensnoop-bpfcc -p "$(pgrep -f myapp)"     # ファイル open
sudo tcpconnect-bpfcc -p "$(pgrep -f myapp)"    # TCP connect
sudo funclatency-bpfcc 'libc:read' -p "$(pgrep -f myapp)"  # syscall latency 分布
```

### tcpdump (パケット観察)

```bash
# tcpdump: 特定ポート + 特定ホスト
sudo tcpdump -i any -w /tmp/cap.pcap 'port 8080 and host upstream.example.com'

# 循環バッファ運用 (10MB × 5 ファイル循環)
sudo tcpdump -i any -w /tmp/cap-%Y%m%d-%H%M%S.pcap -C 10 -W 5 -G 3600 \
  'port 8080 and host upstream.example.com'

# K8s で CNI veth pair を見る
sudo tcpdump -i veth123abc -w /tmp/cap.pcap 'port 8080'
# CNI 別の interface 命名: veth* (kindnet) / cali* (Calico) / cilium_* (Cilium)

# pcap を tshark で要約 (display filter は HTTP 平文のみ、HTTPS は復号鍵が要る)
tshark -r /tmp/cap.pcap -Y 'http.response.code >= 500' \
  -T fields -e ip.src -e ip.dst -e http.request.method -e http.request.uri -e http.response.code

# 実出力例:
# 10.0.1.5    10.0.2.10   POST    /api/v1/orders  503
# 10.0.1.5    10.0.2.11   GET     /api/v1/users   500
```

### strace (最終手段、overhead 注意)

```bash
# どうしても strace を使うなら -c (summary) または -e で絞る
sudo strace -c -p "$(pgrep -f myapp)"        # summary、5-10s で十分

# 特定 syscall のみ
sudo strace -f -e trace=network,openat,connect -p "$(pgrep -f myapp)" -o /tmp/strace.log

# Linux 5.3+: --seccomp-bpf flag で overhead を 10x 以上削減
sudo strace --seccomp-bpf -f -e trace=network -p "$(pgrep -f myapp)"

# macOS の dtruss
sudo dtruss -p "$(pgrep -f myapp)" 2>&1 | grep -E 'connect|read|write'
```

## 期待出力

- `/tmp/cap.pcap` に pcap-ng 形式で通信が保存 (数十 MB〜)
- tshark の TAB 区切り出力で HTTP 500 を発生させた src/dst/URL
- bpftrace は stdout に行 streaming
- perf record の `perf.data` を flamegraph.pl で SVG 化

## ハマりポイント

### strace の overhead (最重要)

- production で安易に attach すると SLO 違反になる ([brendangregg.com: strace Wow Much Syscall](https://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html))
- `-c` (summary only) で短時間、`-e` で対象 syscall を絞る、Linux 5.3+ なら `--seccomp-bpf` を使う
- まず `perf trace` / `bpftrace` を検討する

### BPF capture filter vs Wireshark display filter

両者は **別文法**:

| 段階 | 文法 | 例 | 取れる field |
|---|---|---|---|
| capture filter (`-f` / 引数) | [BPF (libpcap)](https://www.tcpdump.org/manpages/pcap-filter.7.html) | `'port 8080 and host upstream.example.com'` | L2-L4 と payload offset 指定のみ |
| display filter (`-Y`) | [Wireshark syntax](https://wiki.wireshark.org/DisplayFilters) | `'http.response.code >= 500'` | dissector 経由で L7 (`http.*`, `tls.*`) も |

capture 段階で `http.response.code` のような L7 field は使えない (BPF にそのフィールドが無い)。capture filter は緩く、display filter で絞り込みが正解。production capture で BPF を絞りすぎると後で見たい packet が消える。

### tcpdump 4.99+ の pcap-ng 既定

tcpdump 4.99+ は `-w` で default **pcap-ng** 形式を出力 (拡張子 `.pcap` でも中身は pcap-ng)。libpcap 1.0 系の古いツールで読めない事例あり。互換性が必要なら `-F pcap` を付与。

### `-i any` の linux-cooked-mode 制約

- `-i any` は Linux Cooked Capture v1/v2 (SLL / SLL2) の link layer になり、VLAN tag / link layer の一部情報が失われる
- macOS は元々 `-i any` を持たない (BSD 設計起因)。libpcap 1.10+ / Wireshark 4.0+ で `-i pktap,en0,en1` で複数 IF を束ねる代替がある
- HTTPS は中身が暗号化 → `http.response.code` 等の HTTP display filter は適用されない。本文が必要なら [mitmproxy (事例 5-A)](../05-invisible-traffic/A-mitmproxy.md) か SSLKEYLOGFILE で復号

### setcap の副作用

`setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump` を使う場合:

- file capability が付くと dynamic linker は secure-execution mode に入り `LD_LIBRARY_PATH` / `LD_PRELOAD` を無視
- `eip` は legacy 用、modern binary は `+ep` で十分
- package update で binary 置換されると cap も消える
- container では `securityContext.capabilities.add: ["NET_RAW", "NET_ADMIN"]` を Pod spec に書く方が再現性が高い

### 長時間観察

- 循環バッファ必須 (`-C` `-W` `-G`)
- 長時間 strace は禁止 (overhead 累積)。bpftrace / perf record で短時間 sampling を繰り返す

### 参考

- [Brendan Gregg - BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html)
- [iovisor/bcc tools](https://github.com/iovisor/bcc/tree/master/tools)
- [tcpdump 4.99.0 release notes](https://www.tcpdump.org/index.html#latest-releases)
- [Wireshark Wiki: CaptureFilters](https://wiki.wireshark.org/CaptureFilters) / [DisplayFilters](https://wiki.wireshark.org/DisplayFilters)
