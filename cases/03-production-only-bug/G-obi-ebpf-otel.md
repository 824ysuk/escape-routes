# 事例 3-G: OBI (OpenTelemetry eBPF Instrumentation) で out-of-process 計装

> **対象範囲**: 自社が運用するプロダクション環境への観察に限る。eBPF DaemonSet の配備権限が必要。target pod は一切変更しない。詳細はトップ [README.md](../../README.md#適用範囲-scope)。
> **第三者データ**: 扱わない（自社サービスへの観察のみ）
> **PII**: 一時的に通過する可能性（HTTP route / SQL クエリ等の protocol-level trace に URL パラメータ・クエリ本体が含まれる場合がある。OTLP Collector の attribute フィルタリングで収集範囲を制限する）
> **適法境界**: 自社が運用するサービスへの観察のみ。第三者通信への無権限 attach は電気通信事業法 4 条 / 不正アクセス禁止法 3 条に抵触しうる
> **Disclosure 経路**: N/A（自社インフラ観察のため脆弱性発見性なし）

[E (tcpdump / strace)](E-tcpdump-strace.md) は syscall・packet 単位の観察で、HTTP route や SQL query といった protocol level の情報は見えない。[F (eBPF / bpftrace / perf)](F-ebpf-bpftrace.md) は kernel-side の syscall / CPU flame / disk I/O の system layer 観察に特化する。OBI は eBPF を使いつつも OpenTelemetry pipeline に変換した **protocol-level traces + metrics** (HTTP / HTTP2 / gRPC / SQL / Redis / Kafka) を取得し、target pod のコード・image・config・再起動は一切不要。

## 前提 / install

- **Linux kernel 5.8+** または RHEL-family 4.18+（eBPF backport 版）— 5.8 未満では必要な BPF 機能が不足し、attach が無音で失敗する
- **BTF 必須** — `/sys/kernel/btf/vmlinux` が存在すること（kernel 5.14+ は default 有効、Alpine 系 minimal kernel は要確認）
- observer 側 container: `--privileged` + `--pid=host`（または `--pid=container:<name>`）
- **OTLP Collector endpoint** が別途必要（stdout printer のみで完結させる場合は不要）

```bash
# kernel version 確認
uname -r

# BTF 有効確認
ls /sys/kernel/btf/vmlinux
```

OBI image: `otel/ebpf-instrument`（env var prefix `OTEL_EBPF_*`）  
Standalone Beyla（旧）: `grafana/beyla`（env var prefix `BEYLA_*`、OBI への統合後も維持）

## コード

### Docker — port discovery (OBI)

target app が 8080 で listen している場合。`OTEL_EBPF_TRACE_PRINTER=text` で stdout に span を出力できる（Collector 不要で動作確認に使う）。

```sh
docker run --rm \
  --privileged \
  --pid=host \
  -e OTEL_EBPF_OPEN_PORT=8080 \
  -e OTEL_SERVICE_NAME=my-app \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol:4318 \
  -e OTEL_EBPF_TRACE_PRINTER=text \
  otel/ebpf-instrument:main
```

### Docker — executable 名で discovery (OBI)

port が不定・複数 process を同時に計装したい場合。`*/my-binary` のように glob で実行ファイル名を指定する（`AUTO_TARGET_EXE='*'` は全 process マッチ、sidecar 用途のみ）。

```sh
docker run --rm \
  --privileged \
  --pid=host \
  -e OTEL_EBPF_AUTO_TARGET_EXE='*/my-binary' \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol:4318 \
  otel/ebpf-instrument:main
```

### Kubernetes DaemonSet 最小骨格 (OBI)

全 node の host PID を共有し、対象 binary を auto-target するパターン。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: obi
spec:
  selector:
    matchLabels: { app: obi }
  template:
    metadata:
      labels: { app: obi }
    spec:
      hostPID: true
      serviceAccountName: obi
      containers:
        - name: autoinstrument
          image: otel/ebpf-instrument:main
          securityContext: { privileged: true }
          env:
            - { name: OTEL_EBPF_AUTO_TARGET_EXE, value: '*/goblog' }
            - { name: OTEL_EXPORTER_OTLP_ENDPOINT, value: 'http://otelcol:4318' }
            - { name: OTEL_EBPF_KUBE_METADATA_ENABLE, value: 'true' }
```

### Standalone Beyla（旧、参考）

OBI と env var prefix が異なる。`--pid="container:my-app"` で特定 container に絞る使い方も可能。

```sh
docker run --rm \
  --privileged \
  --pid="container:my-app" \
  -e BEYLA_OPEN_PORT=8080 \
  -e BEYLA_SERVICE_NAME=my-app \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol:4318 \
  -e BEYLA_TRACE_PRINTER=text \
  grafana/beyla:latest
```

## 期待出力

`OTEL_EBPF_TRACE_PRINTER=text` を有効にした場合、起動時に instrumentation pipeline 構築ログが出た後、HTTP request 単位で span が 1 行ずつ stdout に出力される。

```
time=2026-06-19T03:14:02.401Z level=INFO msg="creating instrumentation pipeline"
time=2026-06-19T03:14:02.412Z level=INFO msg="Starting main node"
time=2026-06-19T03:14:02.498Z level=INFO msg="found target process" pid=1234 exec_path=/usr/local/bin/my-binary
2026-06-19 03:14:05.812 (12.31ms) HTTP 200 GET /api/users [127.0.0.1]→[10.0.0.5:8080]
2026-06-19 03:14:05.901 (84.55ms) HTTP 500 POST /api/orders [127.0.0.1]→[10.0.0.5:8080]
```

OTLP exporter を設定した場合、Collector / Tempo / Jaeger の trace 画面で span が観測できる（stdout 出力とは独立して動作する）。

## ハマりポイント

### Kernel 要件 (5.8+ または RHEL 4.18+ backport 版)

`uname -r` が 5.8 未満だと必要な BPF 機能が不足し、attach が無音で失敗する。OBI 公式ドキュメントは `Linux kernel: 5.8+, or RHEL-family Linux 4.18+ with the required eBPF backports` と明記している。

### BTF 必須 (`/sys/kernel/btf/vmlinux` の存在)

kernel 5.14+ では default 有効だが、それ以前や Alpine 系の minimal kernel では無効化されているケースがある。`ls /sys/kernel/btf/vmlinux` で事前確認する。

### 暗号化トラフィックの trace context 伝播制約

TLS 終端後の HTTP payload は eBPF で読めるが、context 伝播（W3C traceparent header の継承）は下流 service が OpenTelemetry SDK 等で受け側計装していないと続かない。out-of-process 計装単独では distributed trace が中継先で途切れる。

### Discovery 設定の取り違え

`OTEL_EBPF_OPEN_PORT`（port 一致）・`OTEL_EBPF_EXECUTABLE_PATH`（path 部分一致）・`OTEL_EBPF_AUTO_TARGET_EXE`（glob）は意味が異なる。間違えると「何もマッチしない」か「全 process をマッチして noise が出る」の両極端になる。`AUTO_TARGET_EXE='*'` は sidecar 用途で全 process マッチを意図したときのみ使う。

### Signal カバレッジ

metrics + traces のみ対応。logs / profiling は現状対象外。logs を出したい場合は別途 fluent-bit 等が必要。

### 「zero-code」 marketing 表現の限界

observer 側で `--privileged` + `--pid=host` + discovery 設定 + Collector endpoint 指定が必要なため、host 側の operational config は必要。「target アプリのコード・image・config・再起動が不要」の意味に限定して扱う。

## 参考

- [OpenTelemetry eBPF Instrumentation (OBI) — First Release Announcement](https://opentelemetry.io/blog/2025/obi-announcing-first-release/)
- [Grafana Beyla Donation to OpenTelemetry](https://grafana.com/blog/opentelemetry-ebpf-instrumentation-beyla-donation/)
- [OBI ドキュメント (opentelemetry.io)](https://opentelemetry.io/docs/zero-code/obi/)
- [OBI Setup — Docker](https://opentelemetry.io/docs/zero-code/obi/setup/docker/)
- [OBI Setup — Kubernetes](https://opentelemetry.io/docs/zero-code/obi/setup/kubernetes/)
- [grafana/beyla (GitHub)](https://github.com/grafana/beyla)
- [Beyla Setup — Docker (grafana.com)](https://grafana.com/docs/beyla/latest/setup/docker/)
