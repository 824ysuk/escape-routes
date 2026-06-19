# 事例 7: 観察ツールが入っていない Pod / container を観察したい

> **対象範囲**: 自社が運用する production 環境への観測に限る。観察手段は kernel capability / Pod SecurityContext に影響しうるため、権限要件を理解した上で選ぶ。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 何が起きるか

production の Pod が distroless / scratch / minimal image で動いており、`kubectl exec` で入っても `sh` / `bash` / `tcpdump` / `nslookup` 等がない:

- `kubectl exec -it <pod> -- sh` → `OCI runtime exec failed: exec: "sh": executable file not found`
- `apt install tcpdump` → パッケージマネージャ非搭載で詰まる
- Pod を full image に切り替えて再起動 → 本番 state が失われ再現できなくなる

「何が起きているかを確かめたい」のに、観察ツールを入れる手段がない。

## 代替手段

| 手段 | 概要 | 侵襲度 |
|---|---|---|
| [A. kubectl debug + netshoot (ephemeral container)](A-kubectl-debug-netshoot.md) | k8s 公式機構で観察 image を後付け attach、netns + processNS 共有、tcpdump → ローカル Wireshark pipe まで | 低（再起動なし） |
| [B. host netns 共有 (docker --network container / nsenter -n)](B-host-netns-share.md) | Docker 単体は `docker run --network container:<name>`、k8s は node 上で `nsenter -t <pid> -n` | 低（再起動なし） |
| [C. node 上 tcpdump -i `<veth>` (CNI veth pair)](C-node-veth-tcpdump.md) | pod 完全 untouched、host 側 CNI veth pair で観察。`kubectl-node-shell` や node ssh で実行 | 極低（pod 側 0 変更） |
| [D. ksniff (kubectl krew plugin)](D-ksniff.md) | downey パターンを `kubectl sniff <pod>` 一発化、static tcpdump 注入 + ローカル Wireshark pipe | 低（plugin 追加） |
| [E. eBPF 系 (Cilium Hubble / Inspektor Gadget / Pixie / Kubeshark)](E-ebpf-daemonset.md) | CNI / data plane から target 触らず L7 flow / DNS / HTTP を cluster-wide 観察 | 低（DaemonSet or CNI 機能） |
| [F. service mesh data-plane (Istio / Envoy tap)](F-service-mesh.md) | mesh 前提なら sidecar Envoy の access log / tap filter で L7 を target 触らず観察 | 低（mesh 前提、target 0 変更） |

## 他手段を選ぶ条件

- **A（kubectl debug）**: k8s 環境で手っ取り早く入りたい。1 回きりの調査。k8s 1.25+ で使える
- **B（host nsenter / docker network）**: Docker Compose 環境 (k8s 外) または node ssh が可能な k8s 環境で、Pod 再起動ゼロを優先したい
- **C（CNI veth pair）**: Pod / node に一切手を加えたくない。host からのパケット観察で十分
- **D（ksniff）**: kubectl debug を毎回叩くのが面倒、wireshark pipe を一発で済ませたい
- **E（eBPF 系）**: cluster 全体を継続的に監視したい。target に何も追加せず L7 観察がしたい
- **F（service mesh）**: Istio 等を既に運用中。request/response log + latency が目的

## 補足

distroless 制約と「Pod 再起動不可」を分けて考えると整理しやすい:
- distroless → 観察ツールが入っていないだけ。**外から見る** 手段 (C / E / F) か **別 image をサイドに置く** 手段 (A / D) で解決
- 再起動不可 → A は ephemeral container で **既存 Pod を再起動しない** 機構。B / C も既存 container を一切止めない

観察ツールが使えるようになった後の tcpdump / tshark の filter / overhead / pcap 取り扱いは [事例 3-E: tcpdump / strace / bpftrace](../03-production-only-bug/E-tcpdump-strace.md) 参照。
