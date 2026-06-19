# 事例 7-E: eBPF 系 cluster-wide 観察 (Cilium Hubble / Inspektor Gadget / Pixie / Kubeshark)

> **対象範囲**: 自社が運用する k8s 環境への観測に限る。DaemonSet / privileged pod の配備権限が必要。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 概要

**target Pod を一切変更せず**、kernel の eBPF 機構を使って network flow / DNS / HTTP を cluster-wide に観察する手法群。
DaemonSet を node 全体に配備（または Cilium CNI に組み込み）し、target image がなんであれ・distroless であれ・再起動なしでリアルタイム観察できる。

長期的な観察や cluster 全体の傾向把握に向く。1 回きりの緊急調査なら A / B / D の方が手順が軽い。

## ツール比較

| ツール | CNI 依存 | 主な観察対象 | UI | 特徴 |
|---|---|---|---|---|
| **Cilium Hubble** | Cilium のみ | L4 flow / L7 HTTP・gRPC・DNS | Web UI + CLI | Cilium に統合、zero-overhead |
| **Inspektor Gadget** | 任意 | tcpdump / DNS / TCP connect / execsnoop / 各種 eBPF gadget | CLI (`kubectl gadget`) | 事前 DaemonSet 配備が必要、多機能 |
| **Pixie** | 任意 | L7 HTTP / gRPC / Redis / MySQL / DNS / profiling | Web UI + CLI | Stirling Labs 製 OSS、auto-instrumentation |
| **Kubeshark** | 任意 | L4-L7 (HTTP1/2/3 / WebSocket / gRPC / Kafka) | Web UI | tcpdump + TLS 復号、リアルタイム traffic viewer |
| **Microsoft Retina** | 任意 | TCP metric / drop / DNS / latency | Prometheus / Grafana | 監視寄り、alerting 向け |

## 前提 / install

### Cilium Hubble (Cilium CNI 使用中の場合)

```bash
# Cilium + Hubble が有効か確認
cilium status | grep Hubble

# 無効なら有効化
cilium hubble enable --ui

# hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
```

### Inspektor Gadget

```bash
# kubectl krew 経由でインストール
kubectl krew install gadget

# DaemonSet を cluster に配備
kubectl gadget deploy

# 確認
kubectl gadget version
```

### Pixie

```bash
# Pixie operator (px CLI) — 最新インストール手順は https://docs.px.dev/installing-pixie/ を参照
bash <(curl -fsSL https://withpixie.ai/install.sh)
px auth login
px deploy    # cluster に Pixie をデプロイ
```

### Kubeshark

```bash
# Kubeshark CLI
sh <(curl -Ls https://kubeshark.co/install)

# または brew
brew install kubeshark/tap/kubeshark
```

## コード

### Cilium Hubble: flow 観察

```bash
# port-forward (Hubble relay)
cilium hubble port-forward &

# 全 flow を see (Ctrl-C で停止)
hubble observe --follow

# 特定 namespace の HTTP のみ
hubble observe --namespace default --protocol http --follow

# 特定 pod 宛ての flow
hubble observe --to-pod default/<pod-name> --follow

# DNS クエリ観察
hubble observe --protocol dns --follow

# drop されたパケット
hubble observe --verdict DROPPED --follow

# Web UI (ブラウザ)
cilium hubble ui
```

### Inspektor Gadget: DNS + TCP connect 観察

```bash
# DNS クエリを監視 (全 node)
kubectl gadget trace dns --namespace default --follow

# TCP connect を監視 (特定 pod)
kubectl gadget trace tcp --namespace default --podname <pod-name> --follow

# tcpdump gadget (パケットキャプチャ)
kubectl gadget trace tcpdump -n default --podname <pod-name>

# execsnoop: container 内のプロセス起動を監視
kubectl gadget trace exec -n default --podname <pod-name> --follow
```

### Kubeshark: L7 traffic viewer

```bash
# cluster 全体を観察してブラウザ UI を起動
kubeshark tap

# 特定 namespace を絞り込み
kubeshark tap -n default

# 特定 pod のみ
kubeshark tap <pod-name> -n default

# ブラウザが開かない場合
kubeshark tap --dry-run        # DaemonSet 配備のみ
open http://localhost:8899     # Web UI に手動アクセス
```

### Pixie: HTTP traffic + latency

```bash
# HTTP traffic (latency / error rate / throughput)
px run px/http_data -sstart_time='-5m'

# DNS 解決確認
px run px/dns_data

# network flow map
px run px/net_flow_graph

# PxL スクリプトで custom query
px run -f my_script.pxl
```

## 期待出力

### Hubble observe

```
TIMESTAMP            SOURCE                          DESTINATION                    TYPE       VERDICT  SUMMARY
Jun 19 12:34:56.789  default/myapp-xxx               kube-system/coredns-yyy:53     dns-query  FORWARDED  DNS Query: A upstream-service.default.svc.cluster.local.
Jun 19 12:34:56.790  kube-system/coredns-yyy         default/myapp-xxx              dns-reply  FORWARDED  DNS Reply: A 10.96.1.100
Jun 19 12:34:56.800  default/myapp-xxx               default/upstream-pod-zzz:8080  TCP        FORWARDED  TCP Flags: SYN
```

### Inspektor Gadget DNS trace

```
NAMESPACE    POD              TYPE      QTYPE  NAME                                    RCODE   ADDRESSES
default      myapp-xxx        OUTGOING  A      upstream-service.default.svc.cluster.local.  NoError  10.96.1.100
```

## ハマりポイント

### Cilium Hubble は Cilium CNI 専用

Flannel / Calico / Kindnet を使っている cluster では Hubble は利用できない。CNI を Cilium に移行するかは cluster 管理者の判断が必要。Cilium を使っているかどうか: `kubectl get pods -n kube-system | grep cilium`

### Inspektor Gadget の DaemonSet 配備は node に BPF kernel 要件がある

Linux kernel 5.4+ 推奨。GKE / EKS / AKS は node OS に依存する。Amazon Linux 2 (kernel 4.14) では一部 gadget が動かない。kernel バージョン確認:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kernelVersion}{"\n"}{end}'
```

### Kubeshark は privileged DaemonSet を全 node に配備する

`kubeshark tap` は内部でトラフィックが大きいほど node CPU / memory を使う。production cluster で `kubeshark tap` を全 namespace に走らせると traffic volume に比例して node 負荷が上がる。最初は特定 namespace / pod を絞り込むか `--dry-run` で DaemonSet 配備のみ行って状況確認してから実行する。

### Pixie は Linux eBPF uprobe を使い Go / Java / Node 等の TLS を復号する

Pixie の HTTP 観察は言語 runtime の uprobe により TLS 内部の平文を取得する。すべての TLS 実装に対応しているわけではない (boringssl は対応、独自 TLS スタックは非対応)。`px/http_data` で空欄が多い場合は言語 / TLS ライブラリを確認する。

### eBPF 系は SecurityContext.privileged が不要なケースが多いが例外あり

eBPF DaemonSet 自体は privileged pod で動くが、観察対象 (target pod) には一切変更を加えない。ただし DaemonSet の配備に cluster-admin 相当の RBAC が必要。read-only cluster では DaemonSet をデプロイできない。

## 参考

- [Cilium Hubble (GitHub)](https://github.com/cilium/hubble)
- [Inspektor Gadget (GitHub)](https://github.com/inspektor-gadget/inspektor-gadget)
- [Pixie (pixie-io/pixie GitHub)](https://github.com/pixie-io/pixie)
- [Pixie ドキュメント (px.dev)](https://docs.px.dev/)
- [Kubeshark (GitHub)](https://github.com/kubeshark/kubeshark)
- [Microsoft Retina (GitHub)](https://github.com/microsoft/retina)
- [CloudRaft: eBPF network observability with Cilium Hubble](https://www.cloudraft.io/blog/ebpf-based-network-observability-using-cilium-hubble)
