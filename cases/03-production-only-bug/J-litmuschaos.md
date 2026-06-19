# 事例 3-J: LitmusChaos で観察から制御再現への軸切替

> **対象範囲**: 自社が運用する Kubernetes クラスタへの controlled fault injection に限る。production 適用は annotation gating (`annotationCheck: "true"`) + 観測時間枠の事前合意が前提。詳細はトップ [README.md](../../README.md#適用範囲-scope)。
> **第三者データ**: 扱わない（自社サービスへの fault injection のみ）
> **PII**: 扱わない（ChaosEngine / ChaosResult CR には fault パラメータと verdict のみ。本番データ本体は通過しない）
> **適法境界**: 自社が運用するクラスタへの fault injection のみ。第三者クラスタへの無権限 chaos 注入は不正アクセス禁止法 3 条に抵触しうる
> **Disclosure 経路**: N/A（自社インフラの制御再現のため脆弱性発見性なし）

> **A〜I との役割分担**: [A (trace replay)](A-trace-replay.md) / [B (profiler)](B-profiler.md) / [C (canary)](C-canary.md) / [D (core dump)](D-core-dump.md) / [E (tcpdump / strace)](E-tcpdump-strace.md) / [F (eBPF / bpftrace)](F-ebpf-bpftrace.md) / [G (OBI)](G-obi-ebpf-otel.md) / [H (Pyroscope)](H-pyroscope-continuous-profiling.md) / [I (Polar Signals / Parca)](I-polar-signals-parca.md) はすべて**観察軸**で、起きた事象を後追いで観測する。LitmusChaos は**制御再現軸**で、network partition / pod kill / node failure / resource starvation 等のインフラ事象を意図的に注入して staging で再現し、fix → 再 chaos で validation するループに組み込む。「A〜I で観察できなかった・再現しなかった」場合に本手段へ切り替える。

## 前提 / install

- **Kubernetes クラスタが必須** — k8s 外（VM / serverless）では動作しない。Gremlin (SaaS) / Chaos Mesh (CNCF Incubating) を検討する
- **Litmus 3.x (ChaosCenter)** — Litmus 1.x / 2.x とは architecture が異なる。旧 docs の CR スキーマを流用すると schema mismatch になる
- **`kubectl` + `helm`** が手元に必要

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
kubectl create ns litmus
helm install chaos litmuschaos/litmus --namespace=litmus \
  --set portal.frontend.service.type=NodePort
```

ChaosCenter UI は `kubectl get svc -n litmus` で NodePort を確認してアクセスする。

## コード

### RBAC (`chaos-rbac.yaml`)

ChaosEngine 実行に必要な最小権限。`events` / `configmaps` / `chaosresults` verb が不足すると helper pod が permission error で失敗する（ChaosEngine 自体は `engineState: active` のまま残り、気付きにくい）。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-delete-sa
  namespace: default
  labels:
    app.kubernetes.io/part-of: litmus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-delete-sa
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods", "events", "configmaps"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "deletecollection"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
    verbs: ["list", "get"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "list", "get", "delete", "deletecollection"]
  - apiGroups: ["litmuschaos.io"]
    resources: ["chaosengines", "chaosexperiments", "chaosresults"]
    verbs: ["create", "list", "get", "patch", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-delete-sa
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-delete-sa
subjects:
  - kind: ServiceAccount
    name: pod-delete-sa
    namespace: default
```

### ChaosEngine (`chaosengine.yaml`) — pod-delete experiment

`annotationCheck: "false"` は staging 用設定。production cluster では `"true"` に変更し、target deployment に `litmuschaos.io/chaos="true"` annotation を付けて対象を明示的にゲートする。

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: engine-nginx
  namespace: default
spec:
  engineState: "active"
  annotationCheck: "false"
  appinfo:
    appns: "default"
    applabel: "app=nginx"
    appkind: "deployment"
  chaosServiceAccount: pod-delete-sa
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "15"
            - name: FORCE
              value: "true"
```

### 投入手順

```bash
# RBAC を先に適用
kubectl apply -f chaos-rbac.yaml

# ChaosExperiment CR を hub から取得（version を pin する）
kubectl apply -f https://hub.litmuschaos.io/api/chaos/3.30.0?file=charts/generic/pod-delete/experiment.yaml -n default

# ChaosEngine を投入
kubectl apply -f chaosengine.yaml
```

## 期待出力

```text
$ kubectl get chaosengine -n default
NAME           AGE   ENGINESTATE   EXPERIMENT
engine-nginx   42s   active        pod-delete

$ kubectl get chaosresult engine-nginx-pod-delete -n default
NAME                       AGE   EXPERIMENT   RESULT   VERDICT
engine-nginx-pod-delete    75s   pod-delete   Pass     Pass

$ kubectl logs -n default -l name=pod-delete | tail -3
time="2026-06-20T01:02:03Z" level=info msg="[Chaos]: Killing pod: nginx-7c5f8d-r2qkv"
time="2026-06-20T01:02:18Z" level=info msg="[Status]: Application pod recovered successfully"
time="2026-06-20T01:03:00Z" level=info msg="[Verdict]: Pass"
```

`VERDICT: Pass` は「fault injection 後にアプリが期待通り回復した」ことを示す。`Fail` の場合は再現に成功しているので、そのまま fix 作業に移れる。

## ハマりポイント

### Litmus 3.x で architecture が ChaosCenter 中心に変わった

Litmus 1.x / 2.x の docs を引いて CR を書くと schema が一致しない（Workflow / ChaosEngine の関係性も再構成）。最新 3.x docs ([docs.litmuschaos.io](https://docs.litmuschaos.io/docs/getting-started/installation)) または `hub.litmuschaos.io/api/chaos/<version>?...` の chart を一次情報とする。version を pin しておくと minor update での schema breaking change を回避できる。

### RBAC 不足で Pass しない

pod-delete でも `events` / `configmaps` / `chaosresults` の verb が必要。`kubectl logs` の helper pod 側で permission error が出るが、ChaosEngine 自体は `engineState: active` のまま残るため `kubectl describe chaosengine` または helper pod logs まで掘らないと気付かない。

### namespace 分離

ChaosEngine は同 namespace の ChaosExperiment CR を探す。`appinfo.appns` を別 namespace にしても、ChaosExperiment CR は ChaosEngine と同じ ns に置かないと `experiment not found` で停止する。

### 本番 pod の誤射防止 = annotationCheck

`annotationCheck: "true"` にすると target deployment に `litmuschaos.io/chaos="true"` annotation がない限り fault を注入しない。production cluster では `"false"` のままにせず annotation gating を有効化する。

### k8s 必須、商用代替

declarative CR ベースで運用前提は k8s が必須。k8s 外（VM / serverless）では [Gremlin](https://www.gremlin.com/)（SaaS）/ [Chaos Mesh](https://chaos-mesh.org/)（CNCF Incubating）を検討する。

### 参考

- [LitmusChaos](https://litmuschaos.io/)
- [LitmusChaos — GitHub](https://github.com/litmuschaos/litmus)
- [CNCF — Litmus project](https://www.cncf.io/projects/litmus/)
- [LitmusChaos docs — Getting Started: Installation](https://docs.litmuschaos.io/docs/getting-started/installation)
- [pod-delete experiment](https://litmuschaos.github.io/litmus/experiments/categories/pods/pod-delete/)
- [LitmusChaos Helm chart](https://litmuschaos.github.io/litmus-helm/)
- [Chaos Mesh (商用代替)](https://chaos-mesh.org/)
