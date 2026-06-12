# 事例 4-B: メモリ上限を上げる

## 前提 / install

install 不要、Node / JVM / Go / Ruby が既に install されている前提

## コード

```bash
# Node.js: heap を 8 GB まで拡張
node --max-old-space-size=8192 script.js

# JVM: heap 8 GB、初期 2 GB、OOM 時に heap dump、GC ログ
java -Xmx8g -Xms2g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof -Xlog:gc*:file=/tmp/gc.log -jar app.jar

# Go: GOMEMLIMIT で GC をより積極的に走らせる（Go 1.19+）
GOMEMLIMIT=8GiB go run main.go

# Ruby: GC tuning
RUBY_GC_HEAP_GROWTH_FACTOR=1.1 RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR=2.0 ruby script.rb

# Python: resource module でプロセスごとに上限指定
python -c "import resource; resource.setrlimit(resource.RLIMIT_AS, (8 * 1024**3, 8 * 1024**3)); exec(open('script.py').read())"
```

## 期待出力

- 拡張前に OOM していたスクリプトが正常終了
- JVM の `/tmp/gc.log` 例: `[2.345s][info][gc] GC(0) Pause Young (Allocation Failure) 1024M->256M(2048M) 12.345ms`
- 古い Gen が増え続けて拡張上限に到達したら、stream / SQLite への切り替え検討

## ハマりポイント

- 物理メモリを超えた指定は swap で爆遅。`free -h`（Linux）/ `vm_stat`（macOS）で空きを確認
- **コンテナ環境では cgroup memory limit が優先**: Node に 8 GB 与えても cgroup limit が 4 GB なら 4 GB で OOMKilled。`docker run -m 8g` / K8s の `resources.limits.memory` も併せて引き上げる
- `-Xmx` は heap のみ。Metaspace / off-heap / native memory は別途確保される
