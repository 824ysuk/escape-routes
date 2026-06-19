# 事例 7-B: host netns 共有 (docker --network container / nsenter -n)

> **対象範囲**: 自社が運用する環境への観測に限る。node への ssh / sudo 権限が必要。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 概要

target container の **network namespace (netns) に、host 側から別プロセスを入れる**手法。2 系統ある:

1. **Docker 系**: `docker run --network container:<name>` で Docker daemon 経由でサイドカーを起動し、target と同じ netns を共有
2. **kernel 直叩き系**: `nsenter -t <pid> -n` でホスト OS から target container の netns に直接侵入し、ホスト側の tcpdump 等を実行

k8s 環境では node への ssh (または `kubectl-node-shell`) が必要。Docker Compose / 単体 Docker 環境では daemon API で完結する。

## 前提 / install

```bash
# Docker 系: Docker daemon が動いていれば追加 install なし
docker info

# k8s nsenter 系: node ssh 可能か確認
kubectl get nodes
ssh <node-ip>

# k8s nsenter 系の代替 (node ssh 不要): kubectl-node-shell
kubectl krew install node-shell
kubectl node-shell <node-name>

# nsenter は Linux 標準 (util-linux): 通常プリインストール済み
nsenter --version
```

## コード

### Docker 系: `docker run --network container:<name>`

```bash
# 動作中 container の netns に sibling として netshoot を入れる
docker run --rm -it \
  --network container:<container-name-or-id> \
  nicolaka/netshoot

# 入ったあと: target の eth0 / lo が見える
ip addr
tcpdump -i eth0 -nn 'tcp port 8080'

# 特定の DNS resolver 経由で解決確認
drill example.com @8.8.8.8

# TCP ポート疎通
nc -zv upstream-host 8080
```

```bash
# process namespace も共有する場合 (pid 1 = target の init が見える)
docker run --rm -it \
  --network container:<name> \
  --pid container:<name> \
  nicolaka/netshoot

# → ps -ef で target のプロセス一覧が見える
```

### k8s 系: nsenter で container の netns に直接侵入

```bash
# 1. target pod の node を確認
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'

# 2. node ssh (または kubectl-node-shell)
kubectl node-shell <node-name>
# または: ssh <node-ip>

# 3. crictl で container PID を取得
CONTAINER_ID=$(crictl ps | grep <container-name> | awk '{print $1}')
PID=$(crictl inspect --output json "$CONTAINER_ID" | jq '.info.pid')

# 4. target の netns に入る
sudo nsenter -t "$PID" -n -- tcpdump -i eth0 -nn 'tcp port 8080'

# -n のみ (netns だけ共有): host 側の tcpdump binary を使う
# target の inode: /proc/<pid>/ns/net を確認可
ls -la /proc/"$PID"/ns/
```

```bash
# ctr (containerd) 経由で PID 取得する場合
NAMESPACE=$(kubectl get pod <pod-name> -o jsonpath='{.metadata.namespace}')
CONTAINER_ID=$(ctr -n k8s.io containers ls | grep <pod-name> | awk '{print $1}')
PID=$(ctr -n k8s.io tasks ls | grep "$CONTAINER_ID" | awk '{print $2}')
sudo nsenter -t "$PID" -n -- ip addr
```

### Docker + nsenter 組み合わせ (Docker daemon を経由しない)

```bash
# Docker 環境で daemon 経由せず直接 netns に入る (理由: daemon が応答しない場合等)
DOCKER_PID=$(docker inspect --format '{{.State.Pid}}' <container-name>)
sudo nsenter -t "$DOCKER_PID" -n -- tcpdump -i eth0 -nn
```

## 期待出力

### `ip addr` で target の netns が見える

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
2: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
    link/ether 12:34:56:78:9a:bc brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.1.5/24 brd 10.244.1.255 scope global eth0
```

## ハマりポイント

### Docker daemon 経由 vs nsenter 直叩きの権限差

`docker run --network container:<name>` は Docker daemon (root 権限) を介する。一方 `nsenter -t <pid> -n` は呼び出し元 user の権限で `setns(2)` syscall を発行し、**`CAP_SYS_ADMIN` が必要**。rootless Docker 環境や非 root ユーザーでは nsenter が `Operation not permitted` で落ちる。

| 手段 | root / sudo 要件 |
|---|---|
| `docker run --network container:` | Docker daemon が root ならユーザーは Docker group 所属で OK |
| `nsenter -t <pid> -n` | **実行ユーザーに `CAP_SYS_ADMIN` が必要** (多くの場合 sudo) |
| `kubectl node-shell` + nsenter | node-shell は privileged Pod を経由、pod 内では root |

### `--pid container:` は Docker 1.12+、共有 PID namespace は SecurityContext と競合

`--pid container:<name>` で process namespace も共有できるが、target の Pod spec に `shareProcessNamespace: false` (default) が明示されている場合、Docker daemon 側の `--pid` flag は無視されずに競合するわけではない。Docker standalone 環境なら問題なし。k8s 環境では Pod spec の `shareProcessNamespace: true` を変更する必要がある (再起動が発生するため、本事例の前提「再起動なし」と衝突)。

### k8s の CRI が containerd か CRI-O かで PID 取得コマンドが変わる

| CRI | PID 取得コマンド |
|---|---|
| containerd (デフォルト k8s 1.24+) | `crictl inspect --output json <id> \| jq '.info.pid'` または `ctr -n k8s.io tasks ls` |
| CRI-O | `crictl inspect --output json <id> \| jq '.info.pid'` |
| Docker (k8s 1.23 以前のデフォルト) | `docker inspect --format '{{.State.Pid}}' <name>` |

### nsenter で入ったプロセスの resolv.conf は host の /etc/resolv.conf

`nsenter -n` は netns のみ切り替えるため、DNS 解決に使う `resolv.conf` は **host の `/etc/resolv.conf`** になる。k8s の CoreDNS 経由の解決を確認したいなら `-m`（mount namespace）も追加するか、`drill` に CoreDNS IP を直接指定する:

```bash
# mount ns + net ns を両方共有 (resolv.conf も container のものを使う)
sudo nsenter -t "$PID" -n -m -- drill example.com

# または: CoreDNS IP を直指定
sudo nsenter -t "$PID" -n -- drill example.com @10.96.0.10
```

### `kubectl node-shell` は privileged Pod を作成する

`kubectl node-shell` は DaemonSet ベースで node 上に privileged Pod を立ち上げる。Admission Controller が `privileged: true` を拒否するクラスター（PodSecurityPolicy / OPA Gatekeeper 等）では動作しない。その場合は直接 node ssh が必要。

## 参考

- [Docker docs: --network container mode](https://docs.docker.com/engine/reference/run/#network-settings)
- [Linux man: nsenter(1)](https://man7.org/linux/man-pages/man1/nsenter.1.html)
- [kubectl-node-shell (GitHub)](https://github.com/kvaps/kubectl-node-shell)
- [Kubernetes docs: crictl usage](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Prefetch.net: Debugging k8s network with nsenter + tcpdump](https://prefetch.net/blog/2020/08/03/debugging-kubernetes-network-issues-with-nsenter-dig-and-tcpdump/)
