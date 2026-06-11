# 事例 3: 本番でしか起きないバグを調査したい

> Status: 🚧 WIP（Notion 原稿から転記予定）

## 何が起きるか

local や staging で再現しない、production でだけ落ちる / 遅くなる。`docker-compose up` で本番同等の環境を作ろうとしても、データ量・トラフィック・並行性・外部 API の応答が違うため再現コストが膨大。

## 代替手段

| 手段 | 概要 | 侵襲度 |
|---|---|---|
| A. trace ID から request payload を抜いて local replay | 失敗 request の payload を APM から抜き local で同じ入力を再実行 | 低 |
| B. 本番にプロファイラを attach | py-spy / async-profiler を稼働中プロセスに | 中（CPU 数 % のオーバーヘッド） |
| C. canary deploy + 詳細ログ | 1% のトラフィックにだけ debug log を流す | 中（コード変更要） |
| D. core dump + デバッガ | crash 時の core を取り gdb / delve / lldb で attach | 低（事後） |
| E. tcpdump / strace | syscall・パケット単位で挙動を観察 | 中 |

## 実装例

各代替手段の詳細は近日転記。

## 補足

「local で再現する」を諦めて「本番の生きた状態を観察する」に切り替えるとコストが下がる。観察手段はアプリ層（A・B）・OS 層（D・E）・統計層（C）に分かれる。原則論は Google SRE Book の "Effective Troubleshooting" 章が参考になる。
