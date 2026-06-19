# 事例 7-F: service mesh data-plane (Istio / Envoy access log + tap filter)

> **対象範囲**: 自社が運用する k8s 環境への観測に限る。Istio / service mesh 導入済みが前提。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 概要

Istio 等の service mesh を既に運用している場合、sidecar として各 Pod に注入された **Envoy proxy** が L7 traffic を観察する手段になる。target Pod 自体を変更せず、Envoy の access log / admin API / tap filter / Kiali の可視化で HTTP / gRPC の request / response / latency / error が確認できる。

「distroless Pod でツールが入っていない」問題は mesh 環境では Envoy sidecar が代わりに観察を引き受けることで回避できる。ただし **mesh 未導入なら適用不可**。

## 前提 / 確認

```bash
# Istio がインストールされているか確認
istioctl version
kubectl get pods -n istio-system

# target pod に sidecar が注入されているか確認
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'
# "istio-proxy" が含まれていれば注入済み
```

## コード

### 1. access log で HTTP を観察 (最小手順)

```bash
# Envoy sidecar (istio-proxy container) のログを tail
kubectl logs <pod-name> -c istio-proxy -f

# 特定のステータスコードを絞り込む
kubectl logs <pod-name> -c istio-proxy -f | grep '"response_code":"5'

# JSON 形式の access log を jq で整形 (Istio の default は JSON)
kubectl logs <pod-name> -c istio-proxy -f \
  | jq -r '[.start_time, .method, .path, .response_code, .duration, .upstream_cluster] | @csv'
```

### 2. Envoy admin API で config / stats を確認

```bash
# Envoy admin port に port-forward (default: 15000)
kubectl port-forward <pod-name> 15000:15000

# 別ターミナルで cluster stats (upstream 接続の成功/失敗数)
curl -s localhost:15000/stats | grep "upstream_cx_total\|upstream_rq_total\|upstream_rq_5xx"

# listener の設定を確認
curl -s localhost:15000/config_dump | jq '.configs[] | select(.["@type"] | contains("ListenersConfigDump"))'

# endpoint の healthcheck
curl -s localhost:15000/clusters | grep "health_flags"
```

### 3. Envoy tap filter でリアルタイムパケット観察

tap filter は Envoy v1.11+ で利用可能。HTTP request/response を JSON でストリームする。

```bash
# tap filter を一時追加 (curl で Envoy admin API に POST)
# kubectl exec で istio-proxy container に入る (distroless は target のみ)
kubectl exec -it <pod-name> -c istio-proxy -- bash

# tap filter 設定 (admin port 15000 に直接 POST)
curl -X POST localhost:15000/tap \
  -H 'Content-Type: application/json' \
  -d '{
    "config_id": "test_tap",
    "tap_config": {
      "match": {
        "any_match": true
      },
      "output_config": {
        "sinks": [{"streaming_admin": {}}]
      }
    }
  }'
# → stdout に HTTP request/response が JSON でストリーム
```

### 4. Kiali でサービスグラフ + traffic 確認

```bash
# Kiali UI に port-forward
istioctl dashboard kiali

# または手動
kubectl port-forward svc/kiali -n istio-system 20001:20001
open http://localhost:20001
```

Kiali の "Graph" ビューでサービス間の request rate / error rate / latency が視覚化される。

### 5. Jaeger / Zipkin で distributed trace 確認

```bash
# Jaeger UI
istioctl dashboard jaeger

# trace ID が分かっている場合: Jaeger API で取得
curl "http://localhost:16686/api/traces/<trace-id>"
```

## 期待出力

### Envoy access log (JSON 形式)

```json
{
  "start_time": "2024-06-19T12:34:56.789Z",
  "method": "POST",
  "path": "/api/v1/orders",
  "protocol": "HTTP/1.1",
  "response_code": 503,
  "response_flags": "UH",
  "duration": 142,
  "upstream_cluster": "outbound|8080||upstream-service.default.svc.cluster.local",
  "upstream_host": "10.244.1.10:8080",
  "x_forwarded_for": "-",
  "user_agent": "myapp/1.0"
}
```

`response_flags: "UH"` は "upstream unhealthy" の意 (Envoy の [response flags 一覧](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage) 参照)。

## ハマりポイント

### mesh 未導入なら完全に適用不可

Istio sidecar が注入されていない Pod には Envoy がないためこの手段は使えない。「Istio を今から導入して調査する」は設定変更 + Pod 再起動が必要になり、本事例の「再起動なし」前提と衝突する。

### Envoy access log の粒度は L7 (HTTP) のみ

TCP layer の詳細 (TCP RST / SYN retransmit 等) は Envoy access log には出ない。TCP 層を確認したいなら C 手段 (node veth tcpdump) か A 手段 (kubectl debug + tcpdump) が必要。

### `response_flags` の読み方

Envoy の `response_flags` は 2-3 文字の略語で、エラー原因を示す重要な情報:

| flags | 意味 |
|---|---|
| `UH` | upstream no healthy hosts (全 endpoint が unhealthy) |
| `UF` | upstream connection failure (TCP 接続失敗) |
| `UO` | upstream overflow (circuit breaker) |
| `NR` | no route found |
| `URX` | upstream retry exhausted |
| `RL` | rate limited |
| `-` | 正常終了 |

[全リスト: Envoy docs — response flags](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags)

### Envoy tap filter は一時的・再起動で消える

`/tap` admin API で設定した tap は Pod restart / Envoy restart で消える。永続的にキャプチャしたい場合は Istio の `EnvoyFilter` resource で tap filter を追加する（Istio の設定管理範囲になる）。

### Linkerd / Consul Connect でも類似の手段がある

Linkerd では `linkerd tap` コマンドが Envoy tap filter 相当の機能を提供する:

```bash
# Linkerd tap (L7 traffic をリアルタイム表示)
linkerd viz tap pod/<pod-name>

# Namespace 全体
linkerd viz tap ns/default
```

## 参考

- [Istio: Envoy access log format](https://istio.io/latest/docs/tasks/observability/logs/access-log/)
- [Envoy docs: access log response flags](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags)
- [Envoy docs: tap filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/tap_filter)
- [Kiali: service mesh observability](https://kiali.io/)
- [Linkerd viz tap](https://linkerd.io/2.14/reference/cli/viz/)
