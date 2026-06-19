# 事例 2-G: WebDriver BiDi (storage / network module)

> **対象範囲**: 対象は利用者自身のアカウント、または自社が運用するサービスに限る。テスト目的の自動化に限定 — 本番認証 bypass・他人アカウントへの悪用は不正アクセス禁止法 3 条に該当しうる。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

[事例 2-E (ブラウザ拡張)](E-extension.md) の EditThisCookie は per-browser GUI 拡張依存で自動化と相性が悪く、[事例 2-A (Cookie 抽出)](A-cookie-curl.md) も手動操作を前提とする。CDP 直叩き経路は Chromium 系限定。W3C [WebDriver BiDi](https://www.w3.org/TR/webdriver-bidi/) は classic WebDriver の後継として **cross-browser な storage / network protocol level API** を提供し、Firefox/Chrome 共通の経路で cookie capture / network intercept を扱える。

## 前提 / install

### Selenium 4 (Python)

```bash
pip install selenium
# Firefox (GeckoDriver) を使う場合
brew install geckodriver   # macOS
python -c "from selenium.webdriver.firefox.options import Options; o = Options(); o.enable_bidi = True; print('ok')"

# Chrome を使う場合は ChromeDriver も必要
# pip install webdriver-manager  # 自動管理する場合
```

- `selenium >= 4.20` で BiDi storage / network module の高レベル API が安定化 ([Selenium changelog](https://www.selenium.dev/documentation/webdriver/bidi/))
- Firefox: BiDi がデフォルト有効（`enable_bidi = True` で WebSocket 接続が起動）
- Chrome: CDP がデフォルト。`enable_bidi = True` を渡すことで BiDi 経路を有効化

### Puppeteer (Node.js)

```bash
npm install puppeteer
# Chrome / Firefox のどちらかを指定してインストール
npx puppeteer browsers install chrome
# または Firefox BiDi を使う場合
npx puppeteer browsers install firefox
```

- BiDi 経路には `protocol: 'webDriverBiDi'` を `launch()` に渡す（Chrome はデフォルトが CDP なので必須）
- Firefox は BiDi がデフォルトのため `protocol` 省略可

### Playwright (注意: BiDi 非対応)

```
Playwright は WebDriver BiDi protocol を native サポートしていない。
connectOverCDP() はあるが BiDi 経路は提供されない。
Playwright を使う場合は CDP 経路に留まる。
```

## コード

### Selenium Python — `storage.get_cookies` 経由で cookie 取得

```python
from selenium import webdriver
from selenium.webdriver.firefox.options import Options
from selenium.webdriver.common.bidi.storage import (
    BrowsingContextPartitionDescriptor,
    BytesValue,
    CookieFilter,
)

options = Options()
options.enable_bidi = True  # WebSocket connection を有効化
driver = webdriver.Firefox(options=options)

driver.get("https://example.com/")
driver.add_cookie({"name": "session", "value": "abc123"})

descriptor = BrowsingContextPartitionDescriptor(driver.current_window_handle)
cookie_filter = CookieFilter(
    name="session",
    value=BytesValue(BytesValue.TYPE_STRING, "abc123"),
)
result = driver.storage.get_cookies(filter=cookie_filter, partition=descriptor)

for c in result.cookies:
    print(c.name, c.value.value, result.partition_key.source_origin)

driver.quit()
```

import path・シグネチャは [Selenium trunk の bidi_storage_tests.py](https://github.com/SeleniumHQ/selenium/blob/trunk/py/test/selenium/webdriver/common/bidi_storage_tests.py) から確認済み (`CookieFilter` / `BytesValue` / `BrowsingContextPartitionDescriptor` / `driver.storage.get_cookies(filter=..., partition=...)`)。

### Selenium Python — network intercept + auth handler

```python
from selenium.webdriver.common.bidi.network import Request

def before_request(request: Request):
    # phase=beforeRequestSent で受信した request をそのまま継続
    request.continue_request()

driver.network.add_request_handler("before_request", before_request)
# basic-auth 自動応答 (network.continueWithAuth に相当)
auth_id = driver.network.add_auth_handler("user", "password")

driver.get("https://example.com/private")

driver.network.remove_auth_handler(auth_id)
```

シグネチャは [bidi_network_tests.py](https://github.com/SeleniumHQ/selenium/blob/trunk/py/test/selenium/webdriver/common/bidi_network_tests.py) から確認済み (`driver.network.add_request_handler("before_request", callback)` / `add_auth_handler(user, password)`)。

### Puppeteer — WebDriver BiDi 経由で起動

```javascript
import puppeteer from 'puppeteer';

// Chrome は protocol 明示が必要、Firefox は BiDi がデフォルト
const browser = await puppeteer.launch({ protocol: 'webDriverBiDi' });
const page = await browser.newPage();
await page.goto('https://example.com/');

const cookies = await page.cookies();
console.log(JSON.stringify(cookies, null, 2));

await browser.close();
```

`protocol: 'webDriverBiDi'` および unsupported list は [Puppeteer BiDi page](https://pptr.dev/webdriver-bidi) verbatim。

## 期待出力

### Selenium Python — `storage.get_cookies` 結果 (1 行 = 1 cookie)

```
session abc123 https://example.com
```

### Puppeteer — `page.cookies()` 結果

```json
[
  {
    "name": "session",
    "value": "abc123",
    "domain": "example.com",
    "path": "/",
    "expires": -1,
    "size": 14,
    "httpOnly": false,
    "secure": true,
    "sameSite": "Lax"
  }
]
```

### `network.beforeRequestSent` event stream (subscribe 時)

```
beforeRequestSent context=ABC123 request.method=GET request.url=https://example.com/private
authRequired     context=ABC123 response.status=401 response.headers.www-authenticate=Basic realm="..."
responseStarted  context=ABC123 response.status=200 response.url=https://example.com/private
```

## ハマりポイント

### Selenium Python の network 低レベル API は private

`driver.network._add_intercept()` / `_remove_intercept()` のように underscore prefix が付く。高レベル API (`add_request_handler` / `add_auth_handler`) は stable だが、`network.addIntercept` 相当を直接呼ぶ場合は公開 API の安定性が保証されない（内部実装変更の影響を受ける）。

### Puppeteer BiDi は feature parity 未達

CDP-specific な機能 (`Page.createCDPSession()` / `HTTPRequest.resourceType()` / CPU throttling / Coverage / Tracing / offline mode / drag-and-drop / response `buffer()` / `text()` / `content()` 等) は `UnsupportedOperation` で throw する ([Puppeteer BiDi page](https://pptr.dev/webdriver-bidi))。CDP 機能を使う箇所がある場合は BiDi 経路での置き換えに事前確認が必要。

### Playwright は WebDriver BiDi protocol を native サポートしていない

`connectOverCDP()` はあるが BiDi 経路は提供されていない。[Playwright BrowserType docs](https://playwright.dev/docs/api/class-browsertype) に `bidi` の言及がないことを確認済み。Playwright を使う場合は CDP 経路に留まる。

### auth handler は HTTP auth cache を温める

`continueWithAuth` で一度成功すると browser が同一 origin の credentials を cache し、後続の `authRequired` event が発火しない。テストで challenge を毎回確認するなら driver 再起動が必要 ([Selenium 公式 test の `needs_fresh_driver` marker](https://github.com/SeleniumHQ/selenium/blob/trunk/py/test/selenium/webdriver/common/bidi_network_tests.py))。

### Chrome での BiDi 有効化はオプション必須

Chrome は CDP がデフォルトで、Puppeteer 側は `protocol: 'webDriverBiDi'` を渡さないと CDP 経路に落ちる。Firefox は BiDi デフォルトなので protocol 指定なしで動作する。Selenium Python でも Chrome 使用時は `options.enable_bidi = True` を渡すこと。

## 事例 2-A / 2-C / 2-E との使い分け

| | 事例 2-A (Cookie 抽出 + curl) | 事例 2-C (Playwright + pyotp) | 事例 2-E (ブラウザ拡張) | 事例 2-G (WebDriver BiDi) |
|---|---|---|---|---|
| 操作方式 | 手動でブラウザから抽出 | UI 操作を自動化 | ブラウザ常駐して観察 | 外部から WebSocket driver 制御 |
| 対応 browser | Chromium 系 GUI 拡張依存 | Chromium/Firefox (CDP) | Chrome/Edge (MV3) | Firefox / Chrome (BiDi) |
| session 操作 | Cookie 値を手動コピー | storage_state で保存・再利用 | webRequest で URL/header 観察 | storage module で直接 CRUD |
| network intercept | 不可 | 不可 | onCompleted (header/URL のみ) | addIntercept / continueWithAuth |
| 典型用途 | 短期の手動実行 | UI 経由 2FA の完全自動化 | 普段の操作から副次的に収集 | cross-browser の protocol 直操作 |

## 参考

- [W3C WebDriver BiDi](https://www.w3.org/TR/webdriver-bidi/)
- [W3C WebDriver BiDi: storage module](https://w3c.github.io/webdriver-bidi/#module-storage)
- [W3C WebDriver BiDi: network module](https://w3c.github.io/webdriver-bidi/#module-network)
- [MDN: WebDriver BiDi storage](https://developer.mozilla.org/en-US/docs/Web/WebDriver/Reference/BiDi/Modules/storage)
- [Chrome Developers: WebDriver BiDi](https://developer.chrome.com/blog/webdriver-bidi)
- [Selenium: WebDriver BiDi](https://www.selenium.dev/documentation/webdriver/bidi/)
- [Selenium: BiDi network](https://www.selenium.dev/documentation/webdriver/bidi/network/)
- [Selenium trunk: bidi_storage_tests.py](https://github.com/SeleniumHQ/selenium/blob/trunk/py/test/selenium/webdriver/common/bidi_storage_tests.py)
- [Selenium trunk: bidi_network_tests.py](https://github.com/SeleniumHQ/selenium/blob/trunk/py/test/selenium/webdriver/common/bidi_network_tests.py)
- [Puppeteer: WebDriver BiDi](https://pptr.dev/webdriver-bidi)
- [Playwright: BrowserType API](https://playwright.dev/docs/api/class-browsertype)
