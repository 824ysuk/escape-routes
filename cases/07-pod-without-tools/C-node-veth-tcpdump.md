# 事例 7-C: node 上 tcpdump -i `<veth>` (CNI veth pair)

> **対象範囲**: 自社が運用する k8s 環境への観測に限る。node への ssh または kubectl-node-shell が必要。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 概要

**Pod 側を一切変更せず**、node (host) 側で CNI が作成した veth pair 越しにパケットを観察する。
k8s の各 Pod は独自の network namespace を持ち、host 側には 1 つの veth インターフェースが見える。
このインターフェースを直接 `tcpdump` でキャプチャすることで、Pod 内部を触らずにパケットレベルの観察ができる。

Pod に `kubectl exec` で入れない (distroless / ツール非搭載) 場合でも、Pod が完全に起動していれば常に使える。

## 前提 / install

```bash
# node ssh または kubectl-node-shell でアクセスできること
kubectl node-shell <node-name>        # kubectl-node-shell plugin
# または
ssh <node-ip>                          # 直接 ssh

# node 上で tcpdump が使えること
which tcpdump || sudo apt install tcpdump

# jq (PID 取得の補助)
which jq || sudo apt install jq
```

## コード

### 1. target Pod の veth インターフェースを特定

```bash
# pod が乗っている node を確認
kubectl get pod <pod-name> -o wide

# node shell に入る
kubectl node-shell <node-name>

# 方法 1: crictl + nsenter で pod の eth0 の ifindex を取得 → host 側の対応 veth を逆引き
CONTAINER_ID=$(crictl ps --name <container-name> -q | head -1)
PID=$(crictl inspect --output json "$CONTAINER_ID" | jq '.info.pid')

# pod の eth0 に対応する host 側 interface index を取得
IFINDEX=$(sudo nsenter -t "$PID" -n -- ip link show eth0 | awk '/eth0/{print $1}' | sed 's/:.*//')
# eth0@if<N> の <N> が host 側の interface index
PEER_INDEX=$(sudo nsenter -t "$PID" -n -- ip link show eth0 | grep -oP '(?<=@if)\d+')

# host 側で対応 veth を特定
ip link show | grep "^$PEER_INDEX:"
# 例: 42: cali1234abcd@if3: ...  または  42: veth1234abcd@if3: ...
```

```bash
# 方法 2: pod IP から直接逆引き (シンプル)
POD_IP=$(kubectl get pod <pod-name> -o jsonpath='{.status.podIP}')
# pod が載っている node 上で:
ip route get "$POD_IP" | grep dev  # → "dev cali1234abcd" 等が出る
```

### 2. 特定した veth で tcpdump

```bash
# CNI 別の interface 命名例を踏まえて実行
# Calico:    cali<hash> (例: caliabcd1234)
# Cilium:    lxc<hash>  (例: lxc1234abcd) または eth0 on node
# Flannel:   veth<hash> (例: veth1234abcd)
# Kindnet:   veth<hash>

VETH_IF="caliabcd1234"  # 特定した interface 名に合わせる

# 基本: 全パケット観察
sudo tcpdump -i "$VETH_IF" -nn -s 0

# ポート絞り
sudo tcpdump -i "$VETH_IF" -nn 'tcp port 8080 or tcp port 443'

# ファイルに保存して後で tshark 解析
sudo tcpdump -i "$VETH_IF" -nn -w /tmp/pod-cap.pcap 'tcp port 8080'

# stdout に流して手元 Wireshark へ pipe (ssh tunnel 越しも可)
sudo tcpdump -i "$VETH_IF" -nn -w - 'tcp port 8080' | wireshark -k -i -
```

### 3. DNS 観察 (UDP 53 を含める)

```bash
# DNS 解決失敗の診断: CoreDNS への UDP 53 も捕捉
sudo tcpdump -i "$VETH_IF" -nn '(tcp or udp) port 53 or tcp port 8080'
```

### 4. hostNetwork Pod の場合

`hostNetwork: true` な Pod は独自 netns を持たず host の eth0 / eth1 を直接使う。veth pair は存在しない。

```bash
# hostNetwork pod は host の物理 IF を観察
sudo tcpdump -i eth0 -nn "host $POD_IP"
```

## 期待出力

```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on caliabcd1234, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:34:56.789012 IP 10.244.1.5.51234 > 10.96.0.1.443: Flags [S], seq 123456789, ...
12:34:56.789456 IP 10.96.0.1.443 > 10.244.1.5.51234: Flags [S.], seq 987654321, ...
12:34:56.789789 IP 10.244.1.5.51234 > 10.96.0.1.443: Flags [.], ack 1, win 502, ...
```

## ハマりポイント

### CNI 別 interface 命名が異なる

| CNI | host 側 interface 命名 | 確認コマンド |
|---|---|---|
| Calico | `cali<10文字hash>` | `ip link \| grep cali` |
| Cilium | `lxc<12文字hash>` | `ip link \| grep lxc` |
| Flannel | `veth<8文字hash>` | `ip link \| grep veth` |
| Kindnet | `veth<8文字hash>` | `ip link \| grep veth` |
| Weave | `weave` bridge + `vethwepl<hash>` | `ip link \| grep vethwepl` |
| Canal (Flannel+Calico) | `cali*` | `ip link \| grep cali` |

Cilium では bridge を使わず host 側の interface name が `lxc` で始まる点に注意。

### IPVS mode の kube-proxy では Service VIP を経由しないパケットがある

`kube-proxy --mode=ipvs` を使っている環境では、Service VIP (例: `10.96.x.x`) 宛のパケットは IPVS のロードバランサー変換を受け、**実際の DNAT 後の Pod IP** に書き換えられてから veth に届く。veth で観察できるのは変換後の IP になる。

```bash
# IPVS ルールを確認
sudo ipvsadm -Ln | grep <service-ip>

# 変換前後の IP 対応
kubectl get endpoints <service-name> -o wide
```

### Pod が複数 container を持つ場合は netns は共有

同一 Pod の複数 container は **同じ netns** を共有する。veth は Pod 単位で 1 つ。どの container のパケットかを PID で絞り込むには `tcpdump` の `-Z` flag ではなく A 手段 (kubectl debug --target で processNS 共有) が必要。

### pcap の機密取り扱い

`/tmp` に直書きは world-readable で事故源。専用 dir を 700 で作成してキャプチャし、解析後は削除する:

```bash
install -d -m 700 /var/tmp/pcap-$USER
sudo tcpdump -i "$VETH_IF" -w /var/tmp/pcap-$USER/pod.pcap 'tcp port 8080'
# 解析後:
sudo shred -u /var/tmp/pcap-$USER/pod.pcap
```

詳細な pcap 取り扱い規則 → [事例 3-E](../03-production-only-bug/E-tcpdump-strace.md#pcap-の機密取扱い)

## 参考

- [Calico: CNI veth interface naming](https://docs.tigera.io/calico/latest/reference/public-cloud/aws)
- [Cilium: host-side interface (`lxc*`)](https://docs.cilium.io/en/stable/network/concepts/datapath/)
- [Wireshark Wiki: CaptureFilters](https://wiki.wireshark.org/CaptureFilters)
- [Gist: tcpdump from specific k8s pod (veth pattern)](https://gist.github.com/lavie/80f53eeb09bac9fc1223a850f6977046)
- [tcpdump(1) man page](https://www.tcpdump.org/manpages/tcpdump.1.html)
