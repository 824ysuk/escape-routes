# 事例 1: SNS / 動的サイトからデータが取れない

> **対象範囲**: 取得対象は利用者自身のアカウント、または公開アカウントの公開情報に限る。第三者の非公開コンテンツ・大規模収集は Meta / X / TikTok の ToS 違反 + 不正競争防止法。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 何が起きるか

`curl` や HTTP ライブラリで GET すると 403 が返る、あるいは HTML は取れるが投稿・画像のデータが入っていない（テンプレートだけ）。Instagram・X・LinkedIn・TikTok 等が該当する。原因は (1) サーバーサイドのボット検知、(2) JavaScript で投稿データを後から読み込む動的レンダリング、の 2 つ。

## 検知の 6 層 (anti-bot landscape, 2024-2026)

ボット検知は単一の指標ではなく層構造。UA を変えても他の層で検知される。

| 層 | 例 | 主な検知ベンダー |
|---|---|---|
| 1. IP reputation | datacenter ASN / VPN / Tor / residential | Cloudflare, Akamai |
| 2. TLS fingerprint | [JA3](https://github.com/salesforce/ja3) (2017) / [JA4 / JA4+](https://github.com/FoxIO-LLC/ja4) (2023) | Cloudflare, DataDome |
| 3. HTTP/2 fingerprint | SETTINGS / WINDOW_UPDATE / HEADERS の順序と priority | Akamai |
| 4. HTTP/3 / QUIC fingerprint | ALPN 順序・Transport Parameters (`JA4Q`) | Cloudflare (2024-) |
| 5. JS fingerprint | navigator.webdriver / chrome.runtime / WebGL renderer / Canvas / Audio | Cloudflare Turnstile, hCaptcha, DataDome |
| 6. Behavioral | マウス軌跡 / scroll / click / dwell time | Arkose Labs (FunCaptcha), PerimeterX |

「素 curl + UA 偽装」は層 1+2+3 ですべて curl 固有指紋として検知される。Cookie hijack は層 5+6 を bypass するが層 2 は curl のまま。各手段がどの層に対応するかを意識する:

| 手段 | 1 IP | 2 TLS | 3 H2 | 4 H3 | 5 JS | 6 Behavior |
|---|---|---|---|---|---|---|
| A. ブラウザコンソール | ブラウザ | ブラウザ | ブラウザ | ブラウザ | ブラウザ | 人間 |
| B. Cookie + 素 curl | 自前 | **curl 固有** | **curl 固有** | × | bypass (Cookie) | × |
| B+. Cookie + curl_cffi | 自前 | Chrome 模倣 | Chrome 模倣 | × | bypass | × |
| C. 特化 OSS | 自前 | OSS 次第 | OSS 次第 | × | bypass | × |
| D. 公式 API | 不要 | 不要 | 不要 | 不要 | 不要 | 不要 |
| E. Playwright + stealth | 自前 | ブラウザ | ブラウザ | ブラウザ | 部分 bypass | △ |
| E. rebrowser/patchright | 自前 | ブラウザ | ブラウザ | ブラウザ | 多 bypass | △ |
| F. 商用 (ScrapFly ASP 等) | residential | ベンダー | ベンダー | ベンダー | ベンダー | 人手 solver 連携 |
| G. Jetstream (Bluesky) | 不要 | 不要 | 不要 | 不要 | 不要 | 不要 |

## 法的リスクの 3 層

技術的に動く ≠ 適法。各 SNS の運用は 3 層構造で評価する。

| 層 | 内容 | 判例 / 法令 |
|---|---|---|
| 1. 技術層 | robots.txt / rate limit / [Librahack 事件](https://ja.wikipedia.org/wiki/%E5%B2%A1%E5%B4%8E%E5%B8%82%E7%AB%8B%E4%B8%AD%E5%A4%AE%E5%9B%B3%E6%9B%B8%E9%A4%A8%E4%BA%8B%E4%BB%B6) (2010: req/sec を自己抑制せず誤認逮捕に発展) | robots.txt は法的拘束力なし |
| 2. 契約層 | 各サービス ToS / EULA | [hiQ Labs v. LinkedIn 9th Cir. 2022](https://www.eff.org/cases/hiq-v-linkedin) (公開データへの access は CFAA 違反ではない) / [Bright Data v. Meta N.D. Cal. 2024](https://reason.com/volokh/2024/01/24/bright-data-defeats-meta/) (ToS breach は別途認められた) / [Meta v. BrandTotal 2022](https://casetext.com/case/meta-platforms-inc-v-brandtotal-ltd) (認証済み session は別) |
| 3. データ保護層 | 個人データの収集・処理規制 | EU: [GDPR Art. 5・6](https://gdpr-info.eu/) + Clearview AI €20M 制裁 / 日本: [改正個人情報保護法](https://www.ppc.go.jp/personalinfo/legal/) 17 条・28 条 / 米国: CCPA/CPRA |

技術的に通っても契約層・データ保護層で違法になりうる。逆に公開データの単純な閲覧は CFAA 違反ではない (層 2 が独立)。

## 代替手段

| 手段 | 概要 | 再現性 | 成功率 (2025 Instagram 例) |
|---|---|---|---|
| [A. ブラウザのコンソールで JS を実行](A-browser-console.md) | DOM + Performance API からメモリ取り出し | 一度きり向け | 100% |
| [B. Cookie + curl + UA 偽装](B-cookie-curl.md) | ブラウザ Cookie 抽出 → curl で送る | session 期限まで | 素 curl: 30-60% / curl_cffi: 80%+ |
| [C. 特化 OSS](C-specialized-oss.md) | yt-dlp / gallery-dl / instaloader | 定期実行可 | cookie あり 70-90% |
| [D. 公式 API / DL 機能](D-official-api.md) | Instagram Graph API、ユーザー設定の DL 機能 | 恒久的 | 自分の Business のみ 100% |
| [E. ヘッドレスブラウザ + stealth](E-headless-stealth.md) | Playwright + stealth / rebrowser-playwright | 定期自動化向け | playwright-stealth: 50% / rebrowser: 80% |
| [F. 商用 scraping API](F-commercial-api.md) | ScrapingBee / ScrapFly / Bright Data ほか | 大量・継続向け（有料） | ASP 系: 95%+ |
| [G. Bluesky Jetstream](G-bluesky-jetstream.md) | WebSocket + JSON でリアルタイム購読 | 恒久的（Bluesky のみ） | 100%（認証不要・Bluesky のみ） |

成功率は対象 SNS と時期で大きく変動する。Instagram は datacenter IP を 2023 以降 block 級扱い、X (Twitter) は mobile proxy がほぼ必須。

## 他手段を選ぶ条件

- **B（Cookie + curl）**: 同じデータを別日に再取得したい、スクリプト化したい。素 curl で 403 になるなら curl_cffi に切替
- **C（特化 OSS）**: そのサービスが OSS でカバーされている、CLI 一発で済ませたい
- **D（公式）**: 自社 / 自分のアカウントで、定期業務として組み込みたい (法的・ToS 観点では第一候補)
- **E（Playwright stealth）**: ログインや無限スクロール等の操作が必要。新規導入なら playwright-stealth ではなく [rebrowser-playwright](https://github.com/rebrowser/rebrowser-patches) / [patchright](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright) を主軸に
- **F（商用 API）**: 大量・継続的、自前運用したくない。proxy 種別 (datacenter / ISP / residential / mobile) を SNS に合わせて選ぶ
- **G（Jetstream）**: Bluesky のリアルタイム feed 追跡・特定アカウントの post 追跡・多 collection 同時 subscribe。D（公式 API）の polling ではなく push が必要なとき。archival / 法的証跡が必要な場合は event が unsigned のため不向き（official CBOR firehose を使う）

## 補足

A が最速で動くのは「アカウントは自分」「1 回きりで良い」「ブラウザが既に開いていた」状況。定期実行が必要なら C か D、操作が複雑なら E、コストで殴れるなら F が合う。法的・ToS 観点では D (公式 API) が第一候補で、A/C/E は短期 PoC / 自分のアカウント限定での利用に限る。

検証用 URL (stealth が緑判定になっているか): [bot.sannysoft.com](https://bot.sannysoft.com) / [pixelscan.net](https://pixelscan.net) / [iphey.com](https://iphey.com)
