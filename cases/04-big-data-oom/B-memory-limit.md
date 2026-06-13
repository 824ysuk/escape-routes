# 事例 4-B: メモリ上限を上げる

## 前提 / install

install 不要、Node / JVM / Go / Ruby が既に install されている前提

## コード

### Node.js

```bash
# heap を 8 GB まで拡張
node --max-old-space-size=8192 script.js
```

- `--max-old-space-size` は V8 **old generation のみ**。Buffer (off-heap) や ArrayBuffer は別管理
- 巨大バイナリ扱う場合は `process.memoryUsage()` の `external` / `arrayBuffers` も観測
- `--max-semi-space-size` (young gen) も性能に効く

### JVM (container 配慮)

```bash
# 固定値 (legacy)
java -Xmx8g -Xms2g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof \
  -Xlog:gc*:file=/tmp/gc.log -jar app.jar

# container-aware (推奨): cgroup memory limit に追従
java -XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0 \
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof \
  -Xlog:gc*:file=/tmp/gc.log -jar app.jar
```

- **JDK 8u191+ / JDK 11+ では `-XX:+UseContainerSupport` がデフォルト ON** (明示不要)
- `-XX:MaxRAMPercentage=75.0` で cgroup limit の 75% を heap に。container resize 時に自動追従
- AWS ECS / K8s で `-Xmx` 固定値はアンチパターン (container limit と乖離する)

### Go

```bash
# soft limit。超過しても OOMKill されず GC が積極化
GOMEMLIMIT=8GiB go run main.go

# memory limit driven GC モード (GC trigger を memory に切替)
GOGC=off GOMEMLIMIT=8GiB go run main.go

# runtime からの細かい制御
# import "runtime/debug"
# debug.SetMemoryLimit(8 << 30)
```

- Go 1.19+ の soft limit。OS の OOM killer を起こす前に GC を積極化
- `GOMEMLIMIT` 超過時は GC が頻発し latency が悪化 → 後段で stream / DuckDB に切替

### Ruby

```bash
RUBY_GC_HEAP_GROWTH_FACTOR=1.1 RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR=2.0 ruby script.rb
```

### Python (fail-fast 用途)

```bash
# resource module で上限を「課して」fail-fast させる
# (swap で爆遅になる前に MemoryError で落とす用途)
python -c "import resource; resource.setrlimit(resource.RLIMIT_AS, (8 * 1024**3, 8 * 1024**3)); exec(open('script.py').read())"
```

- CPython の heap はデフォルト無制限なので、他言語と異なり「上限を上げる」側ではなく「上限を引き下げる」側として使う
- **macOS では `ValueError: current limit exceeds maximum limit` で失敗する** (Darwin は `RLIMIT_AS` の変更を受け付けない)。Linux 専用

## cgroup memory limit を確認する

container 環境では cgroup が優先。host で `--max-old-space-size` を 8GB に設定しても cgroup limit が 4GB なら 4GB で OOMKilled になる。

```bash
# cgroup v2 (RHEL 9 / Ubuntu 22.04+ / Debian 12+ default)
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/memory.current

# cgroup v1 (legacy)
cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# どちらか確認
stat -fc %T /sys/fs/cgroup/
# "cgroup2fs" → v2、"tmpfs" → v1

# Docker の場合
docker stats <container>     # 実効値

# K8s の場合
kubectl describe pod <pod>   # resources.limits.memory
```

`docker run -m 8g` / K8s の `resources.limits.memory: 8Gi` も併せて引き上げる。

## heap dump 解析

`-XX:+HeapDumpOnOutOfMemoryError` で取った dump は以下で解析する:

```bash
# 実行中の JVM から手動 dump (jcmd 推奨、JDK 11+)
# jmap は JDK 11 で formally deprecated
jcmd <pid> GC.heap_dump /tmp/live.hprof

# 解析: Eclipse MAT (Memory Analyzer Tool) を起動して .hprof を開く
# - Leak Suspects レポート (自動)
# - Dominator Tree で大きい object を特定
# - 代替: VisualVM / IntelliJ Profiler
```

## 期待出力

- 拡張前に OOM していたスクリプトが正常終了
- JVM の `/tmp/gc.log` 例: `[2.345s][info][gc] GC(0) Pause Young (Allocation Failure) 1024M->256M(2048M) 12.345ms`
- 古い Gen が増え続けて拡張上限に到達したら、stream / SQLite / DuckDB への切り替え検討

## ハマりポイント

- 物理メモリを超えた指定は swap で爆遅。`free -h` (Linux) / `vm_stat` (macOS) で空きを確認
- **コンテナ環境では cgroup memory limit が優先**: Node に 8 GB 与えても cgroup limit が 4 GB なら 4 GB で OOMKilled
- `-Xmx` は heap のみ。Metaspace / off-heap / native memory は別途確保される
- `jmap` は JDK 11 で deprecated、JDK 17+ では `jcmd <pid> GC.heap_dump` を使う
- 暫定対応で時間を稼ぐ用途。データが想定の 10 倍になると次の波で破綻する → 早めに [A. stream](A-stream.md) / [C. DuckDB](C-sqlite-duckdb.md) へ移行

### 参考

- [Oracle: java command (MaxRAMPercentage)](https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html)
- [Oracle JDK 8u191 release notes (UseContainerSupport)](https://www.oracle.com/java/technologies/javase/8u191-relnotes.html)
- [Go GC guide: Memory limit](https://tip.golang.org/doc/gc-guide#Memory_limit)
- [Eclipse MAT](https://www.eclipse.org/mat/)
- [cgroup v2 documentation](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
