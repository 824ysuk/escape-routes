# escape-routes

直接ルート（curl / fetch / アプリを動かす）が詰まったときの代替手段カタログ。
1 事例 = 1 ファイルで、各代替手段に「前提 / install → コード → 期待出力 → ハマりポイント」を揃える。

## 適用範囲 (Scope)

本 repo が扱う手法は、いずれも **利用者自身のアカウント・自社が運用するサービス・書面で許可された対象** に限って適用する前提で書かれている。事例には Cookie 抽出 / TLS 復号 / 本番 PII の取り出し / 認証 session の自動化 / scraping bypass 等、他者に向ければ違法または規約違反になりうる手法が含まれる。

- 第三者の通信内容を傍受 → 日本: 電気通信事業法 4 条 (通信の秘密)、米国: Wiretap Act (18 U.S.C. § 2511)、EU: GDPR Art. 5・6・25、UK: Investigatory Powers Act 2016 §3
- 無権限の認証・識別符号窃用・不正ログイン → 日本: 不正アクセス禁止法 3 条・3 条の 2、米国: CFAA (18 U.S.C. § 1030)、UK: Computer Misuse Act 1990 §1
- SNS の自動取得・大量収集 → 各サービス ToS、不正競争防止法 2 条 1 項 7 号、改正個人情報保護法、GDPR (CNIL の Clearview AI 制裁 €20M 等)
- 公開 SNS の scraping は CFAA 違反ではないとする判例 (hiQ Labs v. LinkedIn 9th Cir. 2022) がある一方、ToS breach は別途認められた (Bright Data v. Meta N.D. Cal. 2024)

利用者は自己の管轄法令と対象サービスの ToS を確認する責任を負う。判断に迷う場合は実行しない。

参考: [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) / [ISO/IEC 29147:2018](https://www.iso.org/standard/72311.html)

## 脆弱性に気づいた場合 (Responsible Disclosure)

本 repo の手法を学習目的で実行した結果、対象サービスの脆弱性に到達するケースは現実に起きる。発見した場合は public に投稿する前に以下の経路をたどる。

1. 対象サービスの `https://example.com/.well-known/security.txt` ([RFC 9116](https://datatracker.ietf.org/doc/html/rfc9116)) を確認
2. 対象が bug bounty 運営中なら HackerOne / Bugcrowd 等で報告
3. 日本国内サービスは [JPCERT-CC 脆弱性関連情報届出](https://jvn.jp/report/) → IPA → ベンダ調整
4. coordinated disclosure timeline は [ISO/IEC 29147:2018](https://www.iso.org/standard/72311.html) / [ISO/IEC 30111:2019](https://www.iso.org/standard/69725.html) 準拠の 90 日を default にする

「観察できた」と「公開してよい」は別。

## なぜこの repo か

- 領域内に閉じた裏技集（[awesome-blackmagic](https://github.com/tnfe/awesome-blackmagic) / [Awesome-Hacking](https://github.com/Hack-with-Github/Awesome-Hacking)）と、問題解決の理論書（[How to Solve It (Polya)](https://en.wikipedia.org/wiki/How_to_Solve_It) / [Lateral Thinking (de Bono)](https://en.wikipedia.org/wiki/Lateral_thinking)）は存在するが、その間（**領域横断 × 具体事例**）は手薄。
- 「動いた = 最適」ではない。再現性・学習コスト・コストで使い分けられるように、**採用しなかった手段も併記** する。
- 個人で育てる前提。1 事例 = 1 PR で追加していく。

## 事例一覧

| # | 事例 | 詰まる原因 | 代替手段数 |
|---|---|---|---|
| 1 | [SNS / 動的サイトからデータが取れない](cases/01-sns-dynamic-site/README.md) | ボット検知・JS 動的レンダリング | 7 |
| 2 | [認証必須サービスから定期取得したい](cases/02-authenticated-service/README.md) | OAuth / 2FA / Cookie 同期 | 6 |
| 3 | [本番でしか起きないバグを調査したい](cases/03-production-only-bug/README.md) | local で再現できない | 7 |
| 4 | [巨大データで OOM が出る](cases/04-big-data-oom/README.md) | メモリに収まらない | 5 |
| 5 | [ブラウザに見えない通信を観察したい](cases/05-invisible-traffic/README.md) | モバイル・他アプリ・暗号化通信 | 5 |
| 6 | [CI でしか起きない失敗を調査したい](cases/06-ci-only-failure/README.md) | ephemeral runner で state がもう失われている | 4 |
| 7 | [観察ツールが入っていない Pod / container を観察したい](cases/07-pod-without-tools/README.md) | distroless / minimal image で観察ツール非搭載 | 6 |

## 共通パターン

7 事例から抽出される、代替手段を考えるときの発想軸。

| パターン | 一言 | 該当事例 |
|---|---|---|
| レイヤーを動かす | HTTP → DOM、API → DB、アプリログ → syscall。抽象を上下させる | 1, 3, 5, 6, 7 |
| ブラウザ session を使う | CLI から再現困難な認証を、ブラウザ経由で済ませる | 1, 2 |
| 状態を運んで再現 | local 環境を整える代わりに、本番 / runner の状態を持ってくる | 3, 6 |
| 問題を分割する | 全体を一度に解かない。stream・チャンク・分業 | 4 |
| 間接ルートに切り替える | 直接アクセスを諦め、proxy・観察ツール・API 経由にする | 2, 5, 6, 7 |

## 各事例の最小スキーマ

新規事例を追加するときは以下の構造で書く:

```
タイトル: <事例タイトル>
詰まる原因: <カテゴリ>
直接ルート: <試みた直接アプローチ>
代替手段:
  - 名前 / 前提・install / 最小コード / 期待出力 / ハマりポイント / リンク
採用手段: <どれか>
採用理由: <再現性 or 学習コスト or コスト>
補足: <一行>
```

## Contributing

PR で事例追加・既存事例の改善を歓迎します。詳細は [CONTRIBUTING.md](CONTRIBUTING.md)。

## 参考リソース

### 領域内の裏技・bypass 集
- [tnfe/awesome-blackmagic](https://github.com/tnfe/awesome-blackmagic) — Web 開発の "黒魔法" コレクション（中国語）
- [Hack-with-Github/Awesome-Hacking](https://github.com/Hack-with-Github/Awesome-Hacking) — セキュリティ系の bypass / 攻撃手法集
- [enaqx/awesome-pentest](https://github.com/enaqx/awesome-pentest) — penetration testing リソース集

### 著名 engineer の実例集
- [Julia Evans](https://jvns.ca/) — debugging / systems / networking
- [Brendan Gregg](https://www.brendangregg.com/) — performance / observability。flamegraph 発案者
- [Dan Luu](https://danluu.com/) — システム・実例の深掘り
- [Sean Goedecke](https://www.seangoedecke.com/) — pragmatic engineer のブログ

### 問題解決の古典
- [How to Solve It (Wikipedia)](https://en.wikipedia.org/wiki/How_to_Solve_It) — George Pólya、1945
- [Lateral Thinking (Wikipedia)](https://en.wikipedia.org/wiki/Lateral_thinking) — Edward de Bono、1967
- [The Pragmatic Programmer](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) — David Thomas & Andrew Hunt

### 事例別の一次情報
- 事例 1: [Performance API (W3C)](https://www.w3.org/TR/performance-timeline/) / [Performance API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API) / [Instagram Graph API](https://developers.facebook.com/docs/instagram-platform) / [Bluesky Jetstream](https://docs.bsky.app/blog/jetstream) / [Jetstream GitHub](https://github.com/bluesky-social/jetstream)
- 事例 2: [OAuth 2.0 (RFC 6749)](https://datatracker.ietf.org/doc/html/rfc6749) / [Device Authorization Grant (RFC 8628)](https://datatracker.ietf.org/doc/html/rfc8628) / [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) / [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) / [W3C WebAuthn Level 3](https://www.w3.org/TR/webauthn-3/)
- 事例 3: [Brendan Gregg — Systems Performance (2nd ed)](https://www.brendangregg.com/sysperfbook.html) / [Google SRE Book](https://sre.google/sre-book/table-of-contents/) / [py-spy](https://github.com/benfred/py-spy) / [async-profiler](https://github.com/async-profiler/async-profiler) / [Argo Rollouts AnalysisTemplate](https://argoproj.github.io/argo-rollouts/features/analysis/) / [OpenTelemetry OBI announce](https://opentelemetry.io/blog/2025/obi-announcing-first-release/) / [OBI setup (Docker)](https://opentelemetry.io/docs/zero-code/obi/setup/docker/) / [OBI setup (Kubernetes)](https://opentelemetry.io/docs/zero-code/obi/setup/kubernetes/) / [Grafana Beyla donation](https://grafana.com/blog/opentelemetry-ebpf-instrumentation-beyla-donation/)
- 事例 4: [Designing Data-Intensive Applications](https://dataintensive.net/) / [Node.js Stream Docs](https://nodejs.org/api/stream.html)
- 事例 5: [mitmproxy ドキュメント](https://docs.mitmproxy.org/stable/) / [Wireshark Wiki](https://wiki.wireshark.org/) / [Android Network Security Config](https://developer.android.com/training/articles/security-config)
- 事例 6: [GitHub Actions Debug Logging](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging) / [actions/upload-artifact](https://github.com/actions/upload-artifact) / [Simon Willison — Debugging Actions with tmate](https://til.simonwillison.net/github-actions/debug-tmate) / [nektos/act usage / runners](https://nektosact.com/usage/runners.html)
- 事例 7: [Kubernetes: Debug Running Pods (ephemeral containers)](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/) / [Kubernetes: Ephemeral Containers concept](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) / [nicolaka/netshoot](https://github.com/nicolaka/netshoot) / [downey.io: kubectl debug + tcpdump → Wireshark](https://downey.io/blog/kubernetes-ephemeral-debug-container-tcpdump/) / [ksniff (eldadru/ksniff)](https://github.com/eldadru/ksniff) / [Cilium Hubble](https://github.com/cilium/hubble) / [Inspektor Gadget](https://github.com/inspektor-gadget/inspektor-gadget) / [Pixie](https://github.com/pixie-io/pixie) / [Kubeshark](https://github.com/kubeshark/kubeshark) / [Istio: Envoy access log](https://istio.io/latest/docs/tasks/observability/logs/access-log/)

### 本記事の事例別ツール一覧
- 事例 1: [yt-dlp](https://github.com/yt-dlp/yt-dlp) / [gallery-dl](https://github.com/mikf/gallery-dl) / [instaloader](https://github.com/instaloader/instaloader) / [Playwright stealth](https://github.com/AtuboDad/playwright_stealth) / [rebrowser-playwright](https://github.com/rebrowser/rebrowser-playwright) / [ScrapingBee](https://www.scrapingbee.com/) / [Bright Data](https://brightdata.com/) / [websocat](https://github.com/vi/websocat) / [Jetstream](https://github.com/bluesky-social/jetstream)
- 事例 2: [Playwright](https://playwright.dev/) / [mitmproxy](https://mitmproxy.org/) / [pyotp](https://github.com/pyauth/pyotp) / [EditThisCookie](https://chromewebstore.google.com/detail/editthiscookie/fngmhnnpilhplaeedifhccceomclgfbg) / [Selenium Virtual Authenticator](https://www.selenium.dev/documentation/webdriver/interactions/virtual_authenticator/) / [CDP WebAuthn domain](https://chromedevtools.github.io/devtools-protocol/tot/WebAuthn/)
- 事例 3: [py-spy](https://github.com/benfred/py-spy) / [async-profiler](https://github.com/async-profiler/async-profiler) / [tcpdump](https://www.tcpdump.org/) / [strace](https://strace.io/) / [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) / [delve](https://github.com/go-delve/delve) / [OBI (otel/ebpf-instrument)](https://github.com/grafana/beyla)
- 事例 4: [SQLite](https://www.sqlite.org/) / [DuckDB](https://duckdb.org/) / [GNU parallel](https://www.gnu.org/software/parallel/) / [`--max-old-space-size`](https://nodejs.org/api/cli.html#--max-old-space-sizesize-in-mib) / [Eclipse MAT](https://www.eclipse.org/mat/)
- 事例 5: [Charles Proxy](https://www.charlesproxy.com/) / [Proxyman](https://proxyman.io/) / [Wireshark](https://www.wireshark.org/) / [iOS Web Inspector](https://developer.apple.com/documentation/safari-developer-tools/inspecting-iphone-or-ipad-apps)
- 事例 6: [mxschmitt/action-tmate](https://github.com/mxschmitt/action-tmate) / [actions/upload-artifact](https://github.com/actions/upload-artifact) / [nektos/act](https://github.com/nektos/act) / [catthehacker/docker_images](https://github.com/catthehacker/docker_images) / [gh CLI](https://cli.github.com/)
- 事例 7: [kubectl debug](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/) / [nicolaka/netshoot](https://github.com/nicolaka/netshoot) / [ksniff](https://github.com/eldadru/ksniff) / [kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell) / [Cilium Hubble](https://github.com/cilium/hubble) / [Inspektor Gadget](https://github.com/inspektor-gadget/inspektor-gadget) / [Pixie](https://github.com/pixie-io/pixie) / [Kubeshark](https://github.com/kubeshark/kubeshark) / [Microsoft Retina](https://github.com/microsoft/retina) / [Istio](https://istio.io/) / [Linkerd viz tap](https://linkerd.io/2.14/reference/cli/viz/)

## License

MIT
