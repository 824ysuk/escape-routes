# 事例 1-E: ヘッドレスブラウザ + stealth

## 前提 / install

```bash
# Python 3.8+
pip install playwright playwright-stealth
playwright install chromium
# Linux 実行時の OS 依存（libnss3 等）
playwright install-deps
# Docker 環境では launch 時に args=['--no-sandbox'] を追加
```

## コード

```python
import asyncio
from playwright.async_api import async_playwright
from playwright_stealth import stealth_async

IS_DOCKER = False   # Docker / コンテナなら True

async def run():
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=['--no-sandbox'] if IS_DOCKER else None,
        )
        ctx = await browser.new_context(
            user_agent='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
            viewport={'width': 1280, 'height': 800},
            locale='ja-JP',
        )
        page = await ctx.new_page()
        await stealth_async(page)
        await page.goto('https://www.instagram.com/USERNAME/', wait_until='networkidle')
        # 無限スクロールで投稿読み込み
        for _ in range(5):
            await page.evaluate('window.scrollTo(0, document.body.scrollHeight)')
            await page.wait_for_timeout(1500)
        await page.screenshot(path='debug.png', full_page=True)
        urls = await page.eval_on_selector_all('article img', 'els => els.map(e => e.src)')
        print(f'urls count: {len(urls)}')
        for u in urls[:5]:
            print(u)
        await browser.close()

asyncio.run(run())
```

## 期待出力

- `debug.png`（数百 KB〜数 MB）にページ全体の screenshot
- stdout に `urls count: 30` のような件数 + 先頭 5 件の URL

## ハマりポイント

- Cloudflare Turnstile / hCaptcha が出るサイトでは stealth でも検知される → F（商用 API）に escalate
- `playwright install-deps` 忘れで `libnss3.so` not found エラー
- Docker / コンテナ環境では `args=['--no-sandbox']` 必須
- `playwright-stealth` は更新頻度が低く、Playwright 本体との互換性に注意。代替: [rebrowser-playwright](https://github.com/rebrowser/rebrowser-playwright)
