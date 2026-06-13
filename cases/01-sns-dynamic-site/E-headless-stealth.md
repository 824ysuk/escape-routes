# 事例 1-E: ヘッドレスブラウザ + stealth

> **対象範囲**: 取得対象は利用者自身のアカウントまたは公開情報に限る。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 選択肢の比較 (2024-2026)

playwright-stealth (Python port) は更新頻度低下で `navigator.webdriver` / CDP fingerprint を残す。検知ベンダー (DataDome / Cloudflare BotScore) は 2024 以降 `Runtime.Enable` CDP コマンドや `utility world` 検知も導入しており、stealth プラグインだけでは [bot.sannysoft.com](https://bot.sannysoft.com) で半分赤判定になる。

| 選択肢 | base | 特徴 | 状況 |
|---|---|---|---|
| [rebrowser-playwright](https://github.com/rebrowser/rebrowser-patches) | Playwright | Source patch で `Runtime.Enable` リーク / `pwInitScripts` 等の Playwright 固有痕跡を消す | npm/pip 推奨 |
| [patchright](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright) | Playwright (Python) | rebrowser-patches を Python に持ち込んだ fork | Python 推奨 |
| [playwright-stealth](https://github.com/AtuboDad/playwright_stealth) | Playwright | navigator.webdriver / chrome.runtime / WebGL 等 古典 8 項目を上書き | 古典検知の補助 |
| [puppeteer-extra-plugin-stealth](https://github.com/berstend/puppeteer-extra) | Puppeteer | stealth プラグインの元祖、Node.js | Node.js 系 |
| [undetected-chromedriver](https://github.com/ultrafunkamsterdam/undetected-chromedriver) | Selenium | 老舗、CDP 隠蔽 | 既存 Selenium 資産がある場合 |
| [nodriver](https://github.com/ultrafunkamsterdam/nodriver) | (Selenium 後継) | undetected-chromedriver 作者の最新作 | 最新 |
| [camoufox](https://github.com/daijro/camoufox) | Firefox | 指紋ランダム化に特化、Firefox 系で評価モデルが浅い | 特定サイトで Chrome 系より成功率高 |

## 前提 / install

```bash
# Python 3.8+
# 推奨: patchright (rebrowser-patches の Python port)
pip install patchright
patchright install chromium
playwright install-deps   # Linux: libnss3 等

# 旧 playwright-stealth (古典検知補助の併用):
# pip install playwright playwright-stealth
# playwright install chromium

# Docker 環境では launch 時に args=['--no-sandbox'] を追加
```

## コード

### 推奨: patchright を使う (Python)

```python
import asyncio
from patchright.async_api import async_playwright   # patchright は API 互換

IS_DOCKER = False

async def run():
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=['--no-sandbox'] if IS_DOCKER else None,
        )
        ctx = await browser.new_context(
            user_agent='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36',
            viewport={'width': 1280, 'height': 800},
            locale='ja-JP',
        )
        page = await ctx.new_page()
        # patchright は stealth 自動適用 (rebrowser-patches 同等)
        await page.goto('https://www.instagram.com/USERNAME/', wait_until='domcontentloaded')
        # 必要要素を明示待ち (networkidle は SNS では never-true)
        await page.wait_for_selector('article img', timeout=10_000)
        # 無限スクロールで投稿読み込み
        for _ in range(5):
            await page.evaluate('window.scrollTo(0, document.body.scrollHeight)')
            await page.wait_for_timeout(1500)
        urls = await page.eval_on_selector_all('article img', 'els => els.map(e => e.src)')
        print(f'urls count: {len(urls)}')
        for u in urls[:5]:
            print(u)
        # screenshot は PII / 認証画面が映る可能性があり artifact 保管に注意
        # await page.screenshot(path='debug.png', full_page=True)
        await browser.close()

asyncio.run(run())
```

### 旧: playwright-stealth (古典 8 項目のみ)

```python
# playwright-stealth は古典検知 (navigator.webdriver, plugins, WebGL ほか) しか上書きしない
# Cloudflare / DataDome に対しては 2024 以降単独では弱い
from playwright_stealth import stealth_async
await stealth_async(page)
```

## 期待出力

- stdout に `urls count: 30` のような件数 + 先頭 5 件の URL
- `bot.sannysoft.com` を `page.goto()` して screenshot を確認すると stealth の効きが見える (緑判定が多いほど良い)

## stealth が消すべき検知シグナル 8 項目

各項目を [bot.sannysoft.com](https://bot.sannysoft.com) の判定列と対応させて独立検証できる。

| シグナル | 期待値 | playwright-stealth | rebrowser/patchright |
|---|---|---|---|
| `navigator.webdriver` | `undefined` | ✅ | ✅ |
| `navigator.plugins.length` | `> 0` | ✅ | ✅ |
| `navigator.languages` | 妥当な値 | ✅ | ✅ |
| `chrome.runtime` (object) | 存在 | ✅ | ✅ |
| WebGL renderer | `Google SwiftShader` でない | ✅ | ✅ |
| `permissions.query({name:'notifications'})` | `default` | △ | ✅ |
| `Notification.permission === navigator.permissions` 整合 | 整合 | × | ✅ |
| iframe `contentWindow` / `chrome` prototype 整合 | 整合 | × | ✅ |
| **CDP `Runtime.Enable` リーク** (新検知) | 検知されない | × | ✅ |
| **`pwInitScripts` (Playwright 固有 prop)** | 存在しない | × | ✅ |

## ハマりポイント

### `wait_until='networkidle'` は SNS で永遠に終わらない

Instagram / X / TikTok は WebSocket / SSE / 定期 polling があり `networkidle` がほぼ never-true。30s timeout で fatal になる。Playwright 公式も 2024 のドキュメントで `networkidle` を deprecated 扱いに近い表現に変更。

**修正**: `wait_until='domcontentloaded'` + `await page.wait_for_selector('...', timeout=10_000)` で必要要素を明示待ち。

### CAPTCHA / WAF の判定 cheat sheet

何が出たら escalate (→ F 商用 API) すべきかを判定する:

| 検知ベンダー | 判定シグナル |
|---|---|
| Cloudflare | response header `cf-mitigated: challenge` / Turnstile script tag |
| DataDome | `<script src="https://ct.captcha-delivery.com/...">` / `datadome` cookie |
| PerimeterX | `<script src="https://client.px-cloud.net/...">` |
| Akamai BMP | `_abck` cookie |
| Cloudflare Turnstile | `<div class="cf-turnstile" data-sitekey="...">` |
| hCaptcha | `<iframe src="https://hcaptcha.com/...">` |
| reCAPTCHA v2/v3 | `<script src="https://www.google.com/recaptcha/api.js">` |
| FunCaptcha (Arkose Labs) | `<iframe src="https://*.arkoselabs.com/...">` |

| CAPTCHA 種別 | 突破難度 (2025) | 推奨ルート |
|---|---|---|
| reCAPTCHA v2 image | 中 | 2captcha / CapSolver で 15-30s / $0.001-0.003 |
| reCAPTCHA v3 score | 中 (IP 評価依存) | 住宅 IP + 振る舞い |
| hCaptcha | 中 | 2captcha / CapSolver |
| Cloudflare Turnstile | 中 | CapSolver / NopeCHA で 5-15s |
| FunCaptcha (Arkose Labs) | **高 (人手 solver でも失敗多)** | F (商用 API) に escalate |
| 独自 image OCR | 低 | GPT-4V / Claude 3 vision で 80-95% |

### Docker / コンテナ

- `playwright install-deps` 忘れで `libnss3.so` not found エラー
- Docker / コンテナ環境では `args=['--no-sandbox']` 必須

### screenshot の artifact

- screenshot は認証済み画面の PII (DM / メールアドレス / 通知バッジ) を含み得る
- CI で artifact を public にしない / retention を短縮 (1-3 日)
- 事例 2-C も同様の罠あり (cross-reference)
