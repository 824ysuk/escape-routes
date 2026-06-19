# cases/

1 事例 = 1 directory。各 directory に `README.md`（何が起きるか + 比較表 + 補足）+ 各代替手段ごとの個別ファイル（A-X.md）が配置される。命名規則は `NN-short-name/`（NN は 2 桁連番）。

事例の構造とトーンは [../CONTRIBUTING.md](../CONTRIBUTING.md) を参照。

## 現在の事例

1. [01-sns-dynamic-site/](01-sns-dynamic-site/README.md) — SNS / 動的サイトからデータが取れない
2. [02-authenticated-service/](02-authenticated-service/README.md) — 認証必須サービスから定期取得したい
3. [03-production-only-bug/](03-production-only-bug/README.md) — 本番でしか起きないバグを調査したい
4. [04-big-data-oom/](04-big-data-oom/README.md) — 巨大データで OOM が出る
5. [05-invisible-traffic/](05-invisible-traffic/README.md) — ブラウザに見えない通信を観察したい
6. [06-ci-only-failure/](06-ci-only-failure/README.md) — CI でしか起きない失敗を調査したい
