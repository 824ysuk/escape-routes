# 事例 7-A: kubectl debug + netshoot (ephemeral container)

> **対象範囲**: 自社が運用する k8s 環境への観測に限る。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 概要

k8s 1.25 GA の [ephemeral container](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) 機構を使い、動作中の Pod に観察用コンテナをオンザフライで追加する。target container の **network namespace + process namespace を共有**するため、再起動なしで `tcpdump` / `drill` / `netcat` 等を実行できる。`nicolaka/netshoot` image は tcpdump / drill / nmap / iftop / tshark / grpcurl / strace 等を詰め込んだデファクト観察 image。

## 前提 / install

```bash
# k8s バージョン確認 (1.25+ で GA / 1.23-1.24 は beta / 1.16-1.22 は alpha)
kubectl version --short

# 1.23-1.24 (beta) の場合: feature gate 確認 (managed cluster は control plane アップグレードが先)
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}'

# ローカル Wireshark (optional, D-ksniff でも代替可)
brew install --cask wireshark  # macOS
sudo apt install wireshark     # Debian / Ubuntu
```

- **kubectl 1.25+** 必須 (client / server 両方)
- `--target` 指定時は Pod spec に `shareProcessNamespace: true` が不要 (ephemeral container 機構が自動共有)
- 自己ローカル Wireshark は pipe パターン (下記) 使用時のみ必要

## コード

### 基本: ephemeral container を attach

```bash
# 1. Pod に netshoot を attach (Pod 再起動なし)
kubectl debug -it <pod-name> \
  --image=nicolaka/netshoot \
  --target=<container-name>
# --target: targetのnetns+processNSを共有。省略すると独立sandbox(下記ハマりポイント参照)

# 2. attach 後、観察を実行
# DNS 解決確認 (drill: k8s CoreDNS 経由)
drill example.com @10.96.0.10

# TCP 疎通確認
nc -zv upstream-service 8080

# プロセス確認 (--target で processNS 共有時のみ target の PID が見える)
ps -ef | grep myapp

# パケット観察
tcpdump -i eth0 -nn -s 0 'tcp port 8080'
```

### 応用: tcpdump → ローカル Wireshark にリアルタイム pipe (downey パターン)

```bash
# ephemeral container 内で tcpdump stdout → ローカル Wireshark に直結
kubectl debug -i <pod-name> \
  --image=nicolaka/netshoot \
  --target=<container-name> \
  -- tcpdump -i eth0 -w - 2>/dev/null \
  | wireshark -k -i -

# ポートを絞る場合
kubectl debug -i <pod-name> \
  --image=nicolaka/netshoot \
  --target=<container-name> \
  -- tcpdump -i eth0 -nn -w - 'tcp port 8080 or tcp port 443' 2>/dev/null \
  | wireshark -k -i -
```

### image 選定比較

| image | サイズ | 収録ツール | 用途 |
|---|---|---|---|
| `nicolaka/netshoot` | ~240 MB | tcpdump / drill / nmap / iftop / tshark / grpcurl / strace / curl / ncat / iperf3 | **デファクト、フル観察** |
| `busybox` | ~1 MB | nc / wget / ping / nslookup | 軽量、DNS/TCP 疎通確認のみ |
| `bitnami/kubectl` | ~50 MB | kubectl のみ | cluster API 操作のみ |
| distroless | × | shell すら無い | `--image` には使えない（下記参照） |

## 期待出力

### `kubectl debug` 起動

```
Defaulting debug container name to debugger-xxxxx.
If you don't see a command prompt, try pressing enter.
                    dP            dP                  dP
                    88            88                  88
 88d888b. .d8888b. d8888P .d8888b. 88d888b. .d8888b. .d8888b. d8888P
 88'  `88 88ooood8   88   Y8ooooo. 88'  `88 88'  `88 88'  `88   88
 88    88 88.  ...   88         88 88    88 88.  ...   88.  .88   88
 dP    dP `88888P'   dP   `88888P' dP    dP `88888P'  `88888P'   dP

<container-name>:~#
```

### `drill` で CoreDNS を直接叩く

```
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 12345
;; ANSWER SECTION:
upstream-service.default.svc.cluster.local. 5 IN A 10.96.1.100
;; Query time: 1 msec
;; SERVER: 10.96.0.10
```

### `tcpdump` 最初のパケット

```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:34:56.789012 IP 10.244.1.5.51234 > 10.96.1.100.8080: Flags [S], seq 123456789, win 64240, ...
12:34:56.789456 IP 10.96.1.100.8080 > 10.244.1.5.51234: Flags [S.], seq 987654321, ack 123456790, ...
```

## ハマりポイント

### `--target` を省略するとプロセス namespace を共有しない

`--target=<container-name>` を省略すると、ephemeral container は同一 Pod に追加されるが network namespace のみ共有され、**process namespace は独立する**。`ps -ef` で target のプロセスが見えず、`tcpdump` で「target が送受したパケットのみ」を PID で絞り込めない。必ず `--target` を指定する ([downey.io 記事](https://downey.io/blog/kubernetes-ephemeral-debug-container-tcpdump/) でも同パターン)。

### k8s バージョン要件

| k8s バージョン | ephemeral container の status | 必要な操作 |
|---|---|---|
| 1.25+ | GA (stable) | なし |
| 1.23-1.24 | beta | なし (自動有効) |
| 1.16-1.22 | alpha | feature gate `EphemeralContainers=true` を control plane に付与 |
| < 1.16 | 非対応 | B 手段 (host nsenter) に切り替える |

managed cluster (EKS / GKE / AKS) は control plane バージョンを `kubectl version` で先に確認。

### `--image` に distroless image は指定できない

debug *対象* (`--target`) が distroless なのは正しい。ただし `--image`（観察コンテナのimage）に distroless を指定すると shell が無いため起動直後に終了する。`--image` には必ず `nicolaka/netshoot` / `busybox` 等ツール入り image を指定する。

### Pod の SecurityContext で CAP_NET_RAW / CAP_NET_ADMIN が drop されていると tcpdump が失敗

```
tcpdump: eth0: You don't have permission to capture on that device
(socket: Operation not permitted)
```

SecurityContext に `capabilities.drop: ["ALL"]` + `allowPrivilegeEscalation: false` を強制している環境では、ephemeral container も同じ制約を継承し tcpdump が失敗する。

回避策:
1. **C 手段（CNI veth pair）**: node 側で観察するため Pod の SecurityContext に依存しない
2. **D 手段（ksniff）**: `--privileged` モードで ephemeral container 自身の capability を上書き可 (ただし privileged node の permission が必要)
3. SecurityContext に `capabilities.add: ["NET_RAW", "NET_ADMIN"]` を追加した ephemeral container を手動 patch で投入

### ephemeral container は削除・変更できない

一度 attach した ephemeral container は `kubectl delete` で削除できない（Pod ごと削除しかない）。変更も不可。オプションを間違えた場合は新しい名前で再 attach する。

```bash
# 別名で追加 (--container オプションで名前指定)
kubectl debug -it <pod-name> \
  --image=nicolaka/netshoot \
  --target=<container-name> \
  --container=debug-v2
```

## 参考

- [Kubernetes: Debug Running Pods — ephemeral containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes: Ephemeral Containers concept](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
- [nicolaka/netshoot (GitHub)](https://github.com/nicolaka/netshoot)
- [downey.io: Debug Kubernetes with kubectl debug + tcpdump + Wireshark](https://downey.io/blog/kubernetes-ephemeral-debug-container-tcpdump/)
