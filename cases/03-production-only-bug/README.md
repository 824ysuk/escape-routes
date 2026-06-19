# 事例 3: 本番でしか起きないバグを調査したい

> **対象範囲**: 自社が運用する production 環境への観測に限る。本番 PII を local に持ち出す手段 (A) は GDPR Art. 5 / 25 / 32 / 改正個人情報保護法 20 条の規律対象。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 何が起きるか

local や staging で再現しない、production でだけ落ちる / 遅くなる。`docker-compose up` で本番同等の環境を作ろうとしても、データ量・トラフィック・並行性・外部 API の応答が違うため再現コストが膨大。

## 代替手段

| 手段 | 概要 | 侵襲度 |
|---|---|---|
| [A. trace ID から request payload を抜いて local replay](A-trace-replay.md) | 失敗 request の payload を APM から抜き local で同じ入力を再実行 | 低 |
| [B. 本番にプロファイラを attach](B-profiler.md) | py-spy / async-profiler を稼働中プロセスに | 中（CPU 数 % のオーバーヘッド） |
| [C. canary deploy + 詳細ログ](C-canary.md) | 1% のトラフィックにだけ debug log を流す | 中（コード変更要） |
| [D. core dump + デバッガ](D-core-dump.md) | crash 時の core を取り gdb / delve / lldb で attach | 低（事後） |
| [E. tcpdump / strace](E-tcpdump-strace.md) | syscall・パケット単位で挙動を観察 | 中 |
| [F. eBPF / bpftrace / perf record](F-ebpf-bpftrace.md) | kernel-level の syscall / CPU flame / ディスク I/O を低 overhead で観察 | 低（eBPF verifier 通過のみ） |
| [G. OBI (OpenTelemetry eBPF Instrumentation)](G-obi-ebpf-otel.md) | eBPF で protocol level (HTTP/gRPC/SQL/Redis/Kafka) を OpenTelemetry trace 化、target pod 非改変 | 低（target pod 非改変、observer 側 privileged + hostPID 必要） |
| [H. Pyroscope (continuous profiling)](H-pyroscope-continuous-profiling.md) | always-on で CPU / memory / I/O を line 単位に記録、時間軸 alignment で spike を事後 query | 中（agent overhead 数 %、OTel eBPF profiler 経路は privileged + hostPID 必要） |
| [I. Polar Signals / Parca (eBPF continuous profiling: 商用 + OSS)](I-polar-signals-parca.md) | parca-agent + Parca server (OSS) または Polar Signals (商用 managed) で eBPF continuous profiling。商用 fallback の選択肢 | 中（agent overhead あり、root / CAP_SYS_ADMIN 必須、object storage backend の設計が必要） |
| [J. LitmusChaos (chaos engineering / 制御再現)](J-litmuschaos.md) | declarative CR (ChaosEngine / ChaosExperiment / ChaosResult) で fault injection、観察軸 (A〜I) で捕まらない事象を制御再現に軸を切り替える | 高（意図的 fault 注入、Litmus 3.x は ChaosCenter 制御プレーン + cluster 内 RBAC が必要、production は annotation gating 推奨） |

## 他手段を選ぶ条件

- **A（trace replay）**: 1 件の失敗 request が特定できている、local 環境がそこそこ整っている
- **C（canary）**: 統計的傾向（一定割合で失敗）、1 件より分布が知りたい
- **D（core dump）**: crash する、落ちる瞬間のメモリ・スタックが知りたい
- **E（tcpdump / strace）**: アプリログに何も出ない、OS との境界を疑っている
- **F（eBPF / bpftrace）**: アプリ層では追えない kernel-side の事象（syscall・scheduler・I/O・network）を long-run 観察したい
- **G（OBI / eBPF auto-instrumentation）**: コード / image / config 変更不可・再起動不可、protocol level の HTTP route や SQL query を見たい。kernel 5.8+（または RHEL 4.18+ backport）前提、distributed trace 維持には下流 service 側 OpenTelemetry SDK 計装も必要
- **H（Pyroscope / continuous profiling）**: spike が偶発する・発生タイミングが予測できない、観測の時間軸 alignment が必要、proactive 最適化と reactive incident debug の両用がしたい
- **I（Polar Signals / Parca）**: continuous profiling を欲するが Grafana stack を採用していない、または self-host Pyroscope の ops 負荷を商用 managed (Polar Signals) に逃がしたい
- **J（LitmusChaos / chaos engineering）**: A〜I で観察できなかった・再現しなかった、production 事象が network partition / pod failure / resource starvation 等のインフラ依存だと仮定が立つ、staging で controlled に再現 → fix → 再 chaos で validation したい

## 補足

「local で再現する」を諦めて「本番の生きた状態を観察する」に切り替えるとコストが下がる。観察手段はアプリ層（A・B）・OS 層（D・E）・統計層（C）に分かれる。A〜I で観察できなかった場合は制御再現軸（J）への切り替えを検討する。原則論は Google SRE Book の "Effective Troubleshooting" 章が参考になる。
