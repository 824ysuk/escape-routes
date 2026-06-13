# 事例 3-C: canary deploy + 詳細ログ

## 前提 / install

- Kubernetes クラスタが既にある前提
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/installation/) を install:

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# kubectl plugin（OS / arch 別に DL URL を切り替え）
# macOS Apple Silicon
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-arm64
chmod +x kubectl-argo-rollouts-darwin-arm64
sudo mv kubectl-argo-rollouts-darwin-arm64 /usr/local/bin/kubectl-argo-rollouts
```

- log aggregator: Loki なら `logcli` 必要 → `brew install loki` / GitHub Releases から DL
- debug 用イメージのビルド + push:

```bash
# 既存 Dockerfile に ENV LOG_LEVEL=DEBUG を追加して別タグでビルド + push
docker build -t registry.example.com/api:debug-canary -f Dockerfile.debug .
docker push registry.example.com/api:debug-canary
```

## コード

rollout.yaml (AnalysisTemplate による自動 rollback 込み):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-error-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: error-rate
    interval: 1m
    successCondition: result[0] < 0.01      # 5xx rate が 1% 未満で success
    failureLimit: 3                          # 3 連続 fail で abort
    inconclusiveLimit: 2
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc:9090
        query: |
          sum(rate(http_requests_total{job="{{args.service-name}}",status=~"5..",rollout="canary"}[5m]))
            /
          sum(rate(http_requests_total{job="{{args.service-name}}",rollout="canary"}[5m]))
  - name: latency-p99
    interval: 1m
    successCondition: result[0] < 1.0       # p99 < 1.0s で success
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc:9090
        query: |
          histogram_quantile(0.99,
            sum by (le) (
              rate(http_request_duration_seconds_bucket{job="{{args.service-name}}",rollout="canary"}[5m])
            )
          )
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-server
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 1
      - pause: { duration: 30m }                 # 観察時間枠
      - analysis:
          templates:
          - templateName: canary-error-rate
          args:
          - name: service-name
            value: api-server
      - setWeight: 100
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: registry.example.com/api:debug-canary
        env:
        - name: LOG_LEVEL
          value: DEBUG
```

```bash
kubectl apply -f rollout.yaml
kubectl argo rollouts get rollout api-server --watch

# Loki でログ観察
logcli query --since 1h '{app="api-server", pod=~"api-server-canary-.*"} |= "error"'

kubectl argo rollouts promote api-server   # 問題なければ 100%
kubectl argo rollouts abort api-server     # 問題あれば stable に戻す
```

## 期待出力

- `kubectl argo rollouts get rollout api-server --watch` の出力例（Name / Strategy / Status / Step / SetWeight / ActualWeight が一覧表示）
- Loki / Datadog で canary 版 pod のみの DEBUG ログ

## ハマりポイント

### SRE 観点 (Google SRE Book "Embracing Risk" / "Effective Troubleshooting" 参照)

canary には以下 4 軸が必須:

1. **観測可能な KPI 定義** — latency p99, error rate (5xx), saturation のいずれか以上
2. **error budget からの抜き取り上限** — 月次 SLO 0.1% 違反予算のうち 10-20% を canary に割当
3. **自動 abort 閾値** — 5xx rate > stable の 1.5x または canary p99 > stable の 2x で auto-rollback
4. **blast radius の上限** — 時間 × RPS = exposure budget (例: 30 分 × 1% × 10k RPS = 18 万 req)

`successCondition` / `failureLimit` / `inconclusiveLimit` の 3 軸が AnalysisTemplate の中核。

### マスキング層

- 1% でも DEBUG ログに PII を含む可能性
- **app code 層でマスクする** (log shipper で後段マスクすると app log と APM trace で別実装になり乖離する)。[OpenTelemetry の LogRecordProcessor](https://opentelemetry.io/docs/specs/otel/logs/data-model/) または structlog `processors` で同じ allow-list ロジックを共有
- Loki の `pipeline_stages.replace` は「保険」と位置付ける
- 1% でも total RPS が高ければ絶対数は十分多い (10k RPS なら 100 RPS)

### 参考

- [Argo Rollouts AnalysisTemplate](https://argoproj.github.io/argo-rollouts/features/analysis/)
- [Google SRE Book - Embracing Risk](https://sre.google/sre-book/embracing-risk/)
- [Google SRE Book - Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)
