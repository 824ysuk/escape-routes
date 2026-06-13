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

rollout.yaml:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-server
spec:
  replicas: 10
  strategy:
    canary:
      # trafficRouting なしだと setWeight は実トラフィック分配にならず、
      # 「canary 用 pod を 1% 相当だけ作って Service の selector が両方を拾う」
      # 動作になる (= pod 数比例 / kube-proxy 確率分配)。1% 厳密に分けるには
      # trafficRouting で Istio / NGINX / ALB / SMI / Traefik を指名する。
      canaryService: api-server-canary
      stableService: api-server-stable
      trafficRouting:
        istio:
          virtualService:
            name: api-server
            routes: [primary]
      steps:
      - setWeight: 1
      - pause: { duration: 30m }
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

- `setWeight` を実トラフィック分配に変えるには [`strategy.canary.trafficRouting`](https://argoproj.github.io/argo-rollouts/features/traffic-management/)（Istio / NGINX / ALB / SMI / Traefik 等）が必須。trafficRouting なしだと pod 数比例（replica count ベース）でしか分配されず、「1% に絞る」運用には届かない
- 1% でも DEBUG ログに PII / token / Authorization が混入する。マスク filter を log shipper に設定するだけでは GDPR / CCPA / 個人情報保護法 の最小収集原則を満たしにくい。最低 (a) DEBUG でも body は hash + length のみ、(b) canary 期間の log retention を 72h 以下、(c) log への access を audit log で記録（[GDPR Art.5(1)(c)](https://gdpr-info.eu/art-5-gdpr/) / [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)）
- 1% でも total RPS が高ければ絶対数は十分多い（10k RPS なら 100 RPS）
- 自動 rollback には Argo Rollouts の [AnalysisTemplate](https://argoproj.github.io/argo-rollouts/features/analysis/) を別途定義
