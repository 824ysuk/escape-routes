# 事例 3: 本番でしか起きないバグを調査したい

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

## 他手段を選ぶ条件

- **A（trace replay）**: 1 件の失敗 request が特定できている、local 環境がそこそこ整っている
- **C（canary）**: 統計的傾向（一定割合で失敗）、1 件より分布が知りたい
- **D（core dump）**: crash する、落ちる瞬間のメモリ・スタックが知りたい
- **E（tcpdump / strace）**: アプリログに何も出ない、OS との境界を疑っている

## 補足

「local で再現する」を諦めて「本番の生きた状態を観察する」に切り替えるとコストが下がる。観察手段はアプリ層（A・B）・OS 層（D・E）・統計層（C）に分かれる。原則論は Google SRE Book の "Effective Troubleshooting" 章が参考になる。
