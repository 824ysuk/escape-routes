# 事例 3-I: Polar Signals / Parca で eBPF continuous profiling (商用 + OSS)

> **対象範囲**: 自社が運用する production 環境への観測に限る。agent overhead が稼働 service の SLO に影響する可能性があるため、観測時間枠・overhead budget の合意を取った上で導入する。
> **第三者データ**: 扱わない（自社サービスへの観測のみ）
> **PII**: 扱わない（profile は CPU time / memory allocation の集計データ。関数名・行番号等のシンボルのみを含む。本番データ本体は profile に現れない）
> **適法境界**: 自社が運用するサービスへの観測のみ。第三者サービスへの無権限 agent 挿入は不正アクセス禁止法 3 条に抵触しうる
> **Disclosure 経路**: N/A（自社インフラ観測のため脆弱性発見性なし）

> **H / G / F との役割分担**: [H (Pyroscope)](H-pyroscope-continuous-profiling.md) は Grafana stack 系 OSS の continuous profiling。本ファイルは parca-agent + Parca server (OSS / Apache 2.0) または Polar Signals (商用 managed backend) の経路に特化し、Grafana stack を採用していない環境・商用 managed への ops 負荷オフロードを選択肢に加える。同一 parca-agent が商用 / OSS どちらの backend にも送信できる点で「OSS で頑張る or 商用に逃がす」軸を表す（[事例 1 の F-commercial-api](../01-sns-dynamic-site/F-commercial-api.md) と同じ構造）。[G (OBI)](G-obi-ebpf-otel.md) は eBPF を使いつつ protocol-level の OpenTelemetry trace 化に特化し、profiling は対象外。[F (eBPF / bpftrace / perf)](F-ebpf-bpftrace.md) は kernel-level の syscall / CPU flame を raw 観察する。

## 前提 / install

- **parca-agent**: eBPF profiler。root または CAP_SYS_ADMIN が必須。Linux kernel 5.3+ かつ BTF (BPF Type Format) が有効であること
- **Parca server** (OSS backend): object storage backend を選択。ローカル検証は FILESYSTEM、長期保存は S3 / GCS / Azure Blob 推奨
- **Polar Signals** (商用 managed backend): 別途アカウント登録と bearer token が必要

```bash
# kernel version と BTF 有効確認
uname -r
ls /sys/kernel/btf/vmlinux

# parca / parca-agent は GitHub Releases からバイナリ取得、またはコンテナイメージを使う
# 公式 README に最新の install 手順あり
# https://github.com/parca-dev/parca#installation
# https://github.com/parca-dev/parca-agent#installation
```

## コード

### Parca server (OSS backend) 起動

```bash
# バイナリ build 済み前提 (make build)
./bin/parca --config-path=parca.yaml --http-address=:7070
```

最小 `parca.yaml`（ローカル検証用、object_storage は FILESYSTEM）:

```yaml
object_storage:
  bucket:
    type: FILESYSTEM
    config:
      directory: ./data

scrape_configs:
  - job_name: default
    scrape_interval: 15s
    static_configs:
      - targets: ['127.0.0.1:7070']
```

### parca-agent (eBPF profiler — 各 node 上で実行)

```bash
sudo parca-agent \
  --node=$(hostname) \
  --remote-store-address=parca:7070 \
  --remote-store-insecure
```

### Polar Signals (商用 managed) に送る場合の差分

`--remote-store-address` を Polar Signals endpoint に差し替え、`--remote-store-insecure` を外して bearer token 系 flag を付与する。詳細は [Polar Signals docs](https://www.polarsignals.com/docs) 参照。

```bash
sudo parca-agent \
  --node=$(hostname) \
  --remote-store-address=grpc.polarsignals.com:443 \
  --remote-store-bearer-token=<YOUR_TOKEN>
```

## 期待出力

Parca server 起動時（構造化ログ、形式は実装バージョンに依存）:

```
level=info msg="starting parca server" version=v0.x.y
level=info msg="HTTP server listening" address=:7070
level=info msg="object storage initialized" type=FILESYSTEM directory=./data
level=info msg="scrape manager started" jobs=1
```

parca-agent attach 後、最初の profile 送信時:

```
level=info msg="parca-agent starting" node=ip-10-0-1-23
level=info msg="BPF program loaded" type=perf_event
level=info msg="process discovered" pid=1234 comm=node executable=/usr/bin/node
level=info msg="profile uploaded" labels="node=ip-10-0-1-23,pid=1234" bytes=18472
```

確認: `http://<parca-host>:7070/` の Web UI で flame graph が描画されれば疎通成功。H (Pyroscope) と同様に continuous に profile が送信され続け、過去の spike を事後 query できる。

## ハマりポイント

### 権限 (root / CAP_SYS_ADMIN)

parca-agent は BPF program load のため root または CAP_SYS_ADMIN が必須（公式 README）。Kubernetes DaemonSet で動かす場合は `securityContext.privileged: true` または明示的 capability 付与が必要。

```yaml
# DaemonSet 抜粋
securityContext:
  privileged: true
```

### kernel 要件 (5.3+ かつ BTF 有効)

Linux kernel 5.3+ かつ `/sys/kernel/btf/vmlinux` が存在することが必要（公式 README）。古い LTS distro の標準 kernel では BTF が無効化されていることがある。

```bash
ls /sys/kernel/btf/vmlinux   # 存在すれば BTF 有効
```

### JIT 言語の symbol 解決

JVM / Node.js / Python など JIT-compiled / interpreted 言語は DWARF または perf_map (`/tmp/perf-<pid>.map`) が無いと flame graph の関数名が `[unknown]` に化ける。

- Java: `-XX:+PreserveFramePointer` 等の追加チューニングが必要
- Node.js: `--perf-basic-prof` フラグで `/tmp/perf-<pid>.map` を生成
- Python: 同様に perf_map のサポートを別途構成する

### object storage (FILESYSTEM は local disk に積み続ける)

FILESYSTEM bucket は local disk に profile データを積み続けるため、長期保存目的での self-host Parca にはローテーション設計または S3 / GCS / Azure Blob 等への切替が必要。`parca.yaml` の `object_storage.bucket.type` で切り替える。

### オーバーヘッド表現

eBPF stack sampling は low-overhead な手法だが、定量的な数値は workload・kernel version・sampling frequency の設定によって変動する。本文に特定の数値を期待しないこと。

### 商用 / OSS 切替時の認証差分

Parca server (OSS) への送信では `--remote-store-insecure` を付けて TLS なしで動かせるが、Polar Signals (商用) は TLS + bearer token が前提。flag の組み合わせを取り違えると接続エラーになる。

### 参考

- [Polar Signals](https://www.polarsignals.com/)
- [Why Polar Signals (docs)](https://www.polarsignals.com/docs/why-polar-signals)
- [parca-dev (GitHub org)](https://github.com/parca-dev)
- [parca (server) README](https://github.com/parca-dev/parca)
- [parca-agent (eBPF profiler) README](https://github.com/parca-dev/parca-agent)
