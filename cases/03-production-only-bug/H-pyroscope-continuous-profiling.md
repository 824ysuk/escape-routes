# 事例 3-H: Pyroscope (continuous profiling) で always-on の時間軸 alignment 観測

> **対象範囲**: 自社が運用する production 環境への観測に限る。agent overhead が稼働 service の SLO に影響する可能性があるため、観測時間枠・overhead budget の合意を取った上で導入する。
> **第三者データ**: 扱わない（自社サービスへの観測のみ）
> **PII**: 扱わない（profile は CPU time / memory allocation の集計データ。関数名・行番号等のシンボルのみを含む。本番データ本体は profile に現れない）
> **適法境界**: 自社が運用するサービスへの観測のみ。第三者サービスへの無権限 agent 挿入は不正アクセス禁止法 3 条に抵触しうる
> **Disclosure 経路**: N/A（自社インフラ観測のため脆弱性発見性なし）

> **B / F / G との役割分担**: [B (py-spy / async-profiler)](B-profiler.md) は one-shot サンプリングで spike 発生中に手動 attach が必要。[F (eBPF / bpftrace / perf)](F-ebpf-bpftrace.md) は kernel-level の syscall / scheduler / I/O を raw 観測する。[G (OBI)](G-obi-ebpf-otel.md) は protocol level (HTTP/gRPC/SQL) を OpenTelemetry trace 化する。本ファイルは always-on で CPU / memory profile を継続記録し、**時間軸 alignment で過去の spike を事後に query** する手段に特化する。

## 前提 / install

```bash
# Python (pyroscope-io)
pip install pyroscope-io

# Go (pyroscope-go)
go get github.com/grafana/pyroscope-go

# Node.js (@pyroscope/nodejs)
npm install @pyroscope/nodejs
```

### OTel eBPF profiler (コード変更なし経路)

アプリのコードを変更せず、[OpenTelemetry eBPF profiler](https://github.com/open-telemetry/opentelemetry-ebpf-profiler) distribution を host 側で起動して OTLP で Pyroscope に送る経路。

前提: **Linux (amd64 / arm64) + `privileged: true` + `hostPID: true`** — Fargate / Cloud Run 等の managed container 環境では動作しない。自前 VM / Kubernetes node-level DaemonSet 配置に限られる。

### Pyroscope サーバー

Pyroscope v2 は object storage backend (S3 / GCS / Azure Blob 等) が前提。Docker Compose での最小起動:

```bash
docker run -d \
  --name pyroscope \
  -p 4040:4040 \
  grafana/pyroscope:latest
```

self-host で local volume のみで起動すると restart 時に profile データを失う。production では storage backend を server install と同時に設計する。

## コード

### Python (pyroscope-io)

```python
import pyroscope

pyroscope.configure(
    application_name="my.python.app",
    server_address="http://my-pyroscope-server:4040",
)

def main():
    # 既存アプリのコード
    pass

if __name__ == "__main__":
    main()
```

default では CPU profiling (`oncpu=True`) のみ。memory profiling は default で取得されず、専用の builtin profiler / tracemalloc 連携を別途構成する必要がある。

### Go (pyroscope-go)

```go
package main

import "github.com/grafana/pyroscope-go"

func main() {
    pyroscope.Start(pyroscope.Config{
        ApplicationName: "simple.golang.app",
        ServerAddress:   "http://pyroscope-server:4040",
        Logger:          pyroscope.StandardLogger,
        ProfileTypes: []pyroscope.ProfileType{
            pyroscope.ProfileCPU,
            pyroscope.ProfileAllocObjects,
            pyroscope.ProfileAllocSpace,
        },
    })
    // 既存アプリのコード
}
```

Go agent は `runtime/pprof` を内部で使う。tight loop / GC-heavy app では CPU overhead が無視できない場合がある。`DisableGCRuns: true` で軽減できるが heap profile の精度が落ちる trade-off。

### Node.js (@pyroscope/nodejs)

```javascript
const Pyroscope = require('@pyroscope/nodejs');

Pyroscope.init({
    serverAddress: 'http://pyroscope:4040',
    appName: 'myNodeService',
});

Pyroscope.start();
```

### OTel eBPF profiler (コード変更なし経路)

```yaml
# docker-compose.yml (抜粋)
services:
  ebpf-profiler:
    image: otel/opentelemetry-collector-ebpf-profiler:0.147.0
    privileged: true
    pid: host
    volumes:
      - /proc:/proc
      - /sys/kernel:/sys/kernel
      - /lib/modules:/lib/modules
      - ./ebpf-profiler-config.yaml:/etc/ebpf-profiler-config.yaml
    command:
      - --config=/etc/ebpf-profiler-config.yaml
      - --feature-gates=service.profilesSupport
```

```yaml
# ebpf-profiler-config.yaml (抜粋)
receivers:
  profiling:
    SamplesPerSecond: 97
exporters:
  otlp/pyroscope:
    endpoint: pyroscope:4040
    tls:
      insecure: true
service:
  pipelines:
    profiles:
      receivers: [profiling]
      exporters: [otlp/pyroscope]
```

## 期待出力

Go agent を起動した場合の標準ログ (StandardLogger 経由):

```
2026/06/19 10:00:00 Starting profiling session
2026/06/19 10:00:00 Pyroscope server address: http://pyroscope-server:4040
2026/06/19 10:00:00 application name: simple.golang.app
2026/06/19 10:00:10 sent profile: type=cpu duration=10s
2026/06/19 10:00:10 sent profile: type=alloc_objects duration=10s
2026/06/19 10:00:10 sent profile: type=alloc_space duration=10s
```

Pyroscope server 側 (`http://localhost:4040`) で `application=simple.golang.app` の query が flamegraph として描画され始めれば疎通成功。B (one-shot attach) と異なり、プロセスを再起動しなくても profile が継続的に送信される。

## ハマりポイント

### Go agent と runtime/pprof

- Go agent は `runtime/pprof` を内部で使うため、tight loop / GC-heavy app で CPU overhead が顕在化しうる
- `DisableGCRuns: true` で GC 起因 overhead を抑制できるが heap profile の精度が落ちる → どちらを優先するか事前に合意する
- `ProfileTypes` を絞る（例: `ProfileCPU` のみ）と overhead を最小化できる

### Python SDK の default は CPU profiling のみ

- `pyroscope.configure()` の default は `oncpu=True` (CPU profiling のみ)
- memory profiling は `profile_allocations=True` + builtin allocator hook、または tracemalloc との連携を別途構成する必要がある
- GIL を持つ Python では off-CPU (I/O 待ち) 時間は CPU profile に現れない。off-CPU も見たい場合は `offcpu=True` を追加する（対応 SDK version 確認が必要）

### OTel eBPF profiler の権限・platform 制約

- `privileged: true` + `hostPID: true` + Linux (amd64 / arm64) が必須
- Fargate / Cloud Run / 一般的な managed container 環境では動作しない
- Kubernetes で使う場合は node-level DaemonSet として配置する（Pod per node）
- `--feature-gates=service.profilesSupport` フラグが必要（2026 時点）

### OTel Profiles signal は Alpha

- OTel Profiles signal は Alpha (under active development)
- collector receiver / exporter の compatibility matrix が頻繁に変わり breaking change が発生しうる
- production 投入時は collector / SDK の version を pin して更新時の動作確認を前提にする

### Pyroscope v2 の object storage backend

- Pyroscope v2 server は object storage backend (S3 / GCS / Azure Blob 等) が前提
- local volume のみで起動すると restart 時に profile データを失う
- self-host 時は storage backend の設計 (bucket policy / lifecycle rule / IAM) を server install と同時に決める

### 参考

- [Grafana Pyroscope (GitHub)](https://github.com/grafana/pyroscope)
- [Pyroscope — Go SDK docs](https://grafana.com/docs/pyroscope/latest/configure-client/language-sdks/go_push/)
- [Pyroscope — Python SDK docs](https://grafana.com/docs/pyroscope/latest/configure-client/language-sdks/python/)
- [Pyroscope — Node.js SDK docs](https://grafana.com/docs/pyroscope/latest/configure-client/language-sdks/nodejs/)
- [Pyroscope — OTel eBPF profiler 連携](https://grafana.com/docs/pyroscope/latest/configure-client/opentelemetry/ebpf-profiler/)
- [OpenTelemetry eBPF profiler (GitHub)](https://github.com/open-telemetry/opentelemetry-ebpf-profiler)
