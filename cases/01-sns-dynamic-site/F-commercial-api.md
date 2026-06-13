# 事例 1-F: 商用 scraping API

> **対象範囲**: 商用 scraping API を経由しても対象サービスとの ToS 関係は利用者と対象サービスの間に成立する (ScrapingBee 公式 [ToS](https://www.scrapingbee.com/terms/) も同旨)。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## ベンダー比較

汎用 render と専用 actor で得意領域が違う。

| ベンダー | 強み | 弱み | 価格目安 |
|---|---|---|---|
| [ScrapingBee](https://www.scrapingbee.com/) | 汎用 render、安価、シンプル API | SNS など難易度高サイトは成功率低 (Instagram 50-70%) | $49/月〜 |
| [ScrapFly](https://scrapfly.io/) | `asp=true` (Anti-Scraping Protection) で JA3 spoof / WAF bypass 内蔵 | 価格やや高 | $30/月〜 |
| [Bright Data](https://brightdata.com/products/web-scraper) | 最大規模の住宅 IP / SERP API / Social Media Scraper API 内蔵 | 高価、KYC 必須 | $500/月〜 |
| [Apify](https://apify.com/store) | Actor marketplace、Instagram Scraper 等の既製 Actor | 出来合いに依存 | $49/月〜 |
| [Zyte API](https://www.zyte.com/zyte-api/) | 旧 Scrapinghub、smart proxy mode | 価格やや高 | $100/月〜 |
| [Oxylabs](https://oxylabs.io/) | 住宅 IP に強い | 高価 | $300/月〜 |
| [Smartproxy / Decodo](https://decodo.com/) | 中価格帯 residential | 機能少 | $50/月〜 |

SNS 系では ScrapFly の `asp=true` か Apify の専用 Actor が成功率高。一般 web の render なら ScrapingBee で十分。

## Proxy 種別と SNS の関係

scraping ベンダーは proxy 種別を選べる。SNS は IP の出自で挙動が変わる。

| 種別 | 出自 | 価格目安 | SNS 成功率 |
|---|---|---|---|
| datacenter | AWS / GCP / Hetzner 等 | 安価 ($1-5/GB) | Instagram / X ほぼ全滅 |
| ISP proxy | datacenter のうち住宅 ASN | 中価格 ($5-10/GB) | 50% 程度 |
| residential | 実在家庭 IP (Bright Data / Oxylabs) | 高価 ($5-15/GB) | 80% 程度 |
| mobile (4G/5G) | キャリア IP | 最高価 ($30-50/GB) | Instagram / X で 95%+ |

ScrapFly / Bright Data / Oxylabs はすべての種別を切替可能。

## 前提 / install

- 例として [ScrapingBee](https://www.scrapingbee.com/) で sign up（無料 1,000 credit、有料 $49/月〜）
- API key 取得 → 環境変数 `API_KEY` に保存
- 対象サイトの `robots.txt` と利用規約を確認 (層 1 / 2、下記「法的リスク」参照)
- 環境変数 `TARGET_URL` に取得対象 URL を設定

## コード

```bash
# 標準（JS 実行込み、約 5 credit / call）
curl -G 'https://app.scrapingbee.com/api/v1/' \
  --data-urlencode "api_key=$API_KEY" \
  --data-urlencode "url=$TARGET_URL" \
  --data-urlencode 'render_js=true' \
  -o rendered.html -w 'status=%{http_code}\n'

# 高難易度サイト用（住宅 IP、約 25 credit / call）
curl -G 'https://app.scrapingbee.com/api/v1/' \
  --data-urlencode "api_key=$API_KEY" \
  --data-urlencode "url=$TARGET_URL" \
  --data-urlencode 'render_js=true' \
  --data-urlencode 'premium_proxy=true' \
  --data-urlencode 'country_code=jp' \
  -o rendered.html

# ScrapFly の ASP 例 (高難易度 SNS 向け)
# curl -G 'https://api.scrapfly.io/scrape' \
#   --data-urlencode "key=$SCRAPFLY_KEY" \
#   --data-urlencode "url=$TARGET_URL" \
#   --data-urlencode 'asp=true' \
#   --data-urlencode 'render_js=true' \
#   --data-urlencode 'country=jp'
```

## 期待出力

- 標準出力: `status=200`
- `rendered.html`（数百 KB）に JS 実行後の完全な HTML
- レスポンス header `Spb-Cost` に消費 credit、`Spb-Resolved-Url` に最終 URL、`Spb-Request-Id` で問い合わせ用 ID

## ハマりポイント

### コストと credit

- `render_js=true` = 5 credit、`premium_proxy=true` = 25 credit、`stealth_proxy=true` = 75 credit
- 失敗（status=4xx, 5xx）時は credit を消費しない（refund される）
- レスポンスが空 / 壊れる場合は `js_scenario` で待機戦略を追加

### 法的リスクの 3 層

商用 API を使っても**利用者と対象サービスの ToS 関係は変わらない**。事例 1 README の「法的リスクの 3 層」参照。

- **層 1 (技術)**: `robots.txt` は法的拘束力ではなくマナー。req/sec を自己抑制する判断軸として残す。User-Agent に自己同定情報 (連絡先メール) を入れる選択肢もある ([Librahack 事件](https://ja.wikipedia.org/wiki/%E5%B2%A1%E5%B4%8E%E5%B8%82%E7%AB%8B%E4%B8%AD%E5%A4%AE%E5%9B%B3%E6%9B%B8%E9%A4%A8%E4%BA%8B%E4%BB%B6) 教訓)
- **層 2 (契約)**: ToS / Cease-and-desist letter のほうが robots.txt より優先。[hiQ Labs v. LinkedIn 9th Cir. 2022](https://www.eff.org/cases/hiq-v-linkedin) で公開データ scraping は CFAA 違反ではないと確定したが、[Bright Data v. Meta N.D. Cal. 2024](https://reason.com/volokh/2024/01/24/bright-data-defeats-meta/) では ToS breach は別途認められた
- **層 3 (データ保護)**: EU GDPR (CNIL の Clearview AI 制裁 €20M)、日本 改正個人情報保護法 17・28 条 (要配慮個人情報の取得規制)、米国 CCPA/CPRA
- **residential proxy の倫理**: ethical な提供者 (Bright Data の KYC 経由) と grey な提供者 (911 S5 botnet が 2024 年に米司法省に摘発された) を区別する

### CAPTCHA solver の連携

対象サイトに CAPTCHA が出る場合は商用 API の解法サービスに委譲する:

- [2captcha](https://2captcha.com/api-docs) — 人手 + ML、reCAPTCHA / hCaptcha / Turnstile / FunCaptcha
- [CapSolver](https://capsolver.com/) — ML 中心、Cloudflare Turnstile に強い
- [NopeCHA](https://nopecha.com/) — ML 中心、ブラウザ拡張版あり

Playwright と連携する場合の典型: `page.evaluate(() => document.querySelector('iframe[title="..."]').dataset.sitekey)` で sitekey 取得 → solver API → `g-recaptcha-response` textarea に注入 → form submit。
