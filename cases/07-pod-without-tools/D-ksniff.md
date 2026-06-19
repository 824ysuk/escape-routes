# 事例 7-D: ksniff (kubectl krew plugin)

> **対象範囲**: 自社が運用する k8s 環境への観測に限る。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 概要

[ksniff](https://github.com/eldadru/ksniff) は `kubectl sniff <pod>` 一発で「ephemeral container に static tcpdump を注入 → stdout → ローカル Wireshark へ pipe」を自動化する kubectl krew plugin。[A 手段 (kubectl debug + downey パターン)](A-kubectl-debug-netshoot.md) の手順をスクリプト化したものと考えてよい。

2 モードあり:
- **static モード** (default): static binary の tcpdump を pod にアップロードして実行。distroless 対応。root 権限不要
- **privileged モード** (`-p`): node 上に privileged pod を起動して node 側から観察。pod の SecurityContext に依存しない

## 前提 / install

```bash
# krew のインストール (krew 自体が入っていない場合)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# ksniff インストール
kubectl krew install sniff

# Wireshark (GUI または TUI)
brew install --cask wireshark   # macOS
sudo apt install wireshark      # Debian / Ubuntu
```

## コード

### 基本: static モード (distroless 対応)

```bash
# 最小: pod 全パケットを Wireshark に流す
kubectl sniff <pod-name>

# namespace 指定
kubectl sniff <pod-name> -n <namespace>

# コンテナ指定 (複数コンテナ Pod)
kubectl sniff <pod-name> -c <container-name>

# ファイルに保存 (Wireshark を後で開く)
kubectl sniff <pod-name> -o /tmp/pod.pcap
```

### フィルタを付ける

```bash
# BPF capture filter で絞り込む (tcpdump -f 相当)
kubectl sniff <pod-name> -f 'tcp port 8080'
kubectl sniff <pod-name> -f '(tcp or udp) port 53'
kubectl sniff <pod-name> -f 'host upstream-service and tcp port 443'
```

### privileged モード: SecurityContext が厳しい環境 / DaemonSet ベース観察

```bash
# node 上に privileged pod を建てて観察 (pod 側 SecurityContext を回避)
kubectl sniff <pod-name> -p

# privileged モードで特定フィルタ + ファイル保存
kubectl sniff <pod-name> -p -f 'tcp port 8080' -o /tmp/cap.pcap
```

### tshark と組み合わせてターミナルで見る (Wireshark なし環境)

```bash
# tshark に stdin を流す
kubectl sniff <pod-name> -f 'tcp port 8080' | \
  tshark -r - -T fields -e ip.src -e ip.dst -e tcp.dstport -e http.request.method -e http.request.uri
```

## 期待出力

```
INFO[0000] sniffing container: <container-name>
INFO[0000] no container name provided, using first: <container-name>
INFO[0000] uploading static tcpdump binary to pod...
INFO[0001] tcpdump binary uploaded successfully
INFO[0001] starting remote tcpdump
INFO[0001] starting Wireshark
```

Wireshark が起動してリアルタイムにパケットが表示される。

## ハマりポイント

### static モードと privileged モードの動作原理の差

| 項目 | static モード | privileged モード |
|---|---|---|
| 動作原理 | tcpdump static binary を pod にコピーして実行 | node 上に privileged pod を起動して node 側から sniff |
| distroless 対応 | **対応** (shell 不要な static binary 注入) | 対応 (pod 側を触らない) |
| 必要な権限 | pod へのファイルコピー (`exec`) ができる RBAC | node 上で privileged pod を起動できる権限 |
| SecurityContext `readOnlyRootFilesystem: true` | **非対応** (binary write が必要) | 対応 |
| 観察 granularity | pod netns の eth0 | node の veth 越し (pod 側 IF と同等) |

`readOnlyRootFilesystem: true` の distroless pod は **static モードも使えない**。その場合は privileged モード (`-p`) か C 手段 (node veth tcpdump) に切り替える。

### ksniff のバージョンと k8s API の互換性

ksniff は内部で `ephemeralcontainers` subresource API を使う。k8s 1.25 未満で GA 前の環境では動作しない場合がある。`kubectl sniff --help` で使用 API バージョンを確認。

```bash
kubectl sniff --help 2>&1 | grep -i "kube-version\|k8s\|api"
```

### Wireshark が PATH に入っていない場合

macOS で Wireshark を brew cask でインストールした場合、`wireshark` コマンドが `/Applications/Wireshark.app/Contents/MacOS/Wireshark` に存在するが PATH には入っていない。

```bash
# PATH に追加
export PATH="$PATH:/Applications/Wireshark.app/Contents/MacOS"

# または: ファイル保存 (-o) してから GUI で開く
kubectl sniff <pod> -o /tmp/cap.pcap
open /tmp/cap.pcap  # macOS なら Wireshark で開く
```

### ファイルに保存した pcap の機密取り扱い

`-o /tmp/cap.pcap` は world-readable パスへの書き込み。詳細 → [事例 3-E: pcap の機密取扱い](../03-production-only-bug/E-tcpdump-strace.md#pcap-の機密取扱い)

### ksniff のメンテナンス状況

ksniff は 2023 年以降コミット頻度が低下しており、k8s 1.28+ 環境での動作問題が報告されている（[GitHub Issues](https://github.com/eldadru/ksniff/issues) 参照）。`kubectl sniff` が想定通りに動作しない場合の代替:
- **C 手段 (node 側 veth tcpdump)**: Pod 完全 untouched で同等の観察が可能 → [C-node-veth-tcpdump.md](C-node-veth-tcpdump.md)
- **Inspektor Gadget**: `kubectl gadget trace tcpdump -n <namespace> --podname <pod>` — E 手段のサブセット、任意 CNI 対応 → [E-ebpf-daemonset.md](E-ebpf-daemonset.md)

## 参考

- [ksniff (eldadru/ksniff) GitHub](https://github.com/eldadru/ksniff)
- [Red Hat Blog: ksniff overview](https://www.redhat.com/en/blog/capture-packets-kubernetes-ksniff)
- [kubectl krew: plugin registry](https://krew.sigs.k8s.io/plugins/)
- [downey.io: 手動パターン (ksniff が自動化する手順の原型)](https://downey.io/blog/kubernetes-ephemeral-debug-container-tcpdump/)
