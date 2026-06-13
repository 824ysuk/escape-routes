# 事例 2-C: Playwright で完全自動化

> **対象範囲**: 対象は利用者自身のアカウント、または自社が運用するサービスに限る。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 前提 / install

```bash
pip install playwright pyotp
playwright install chromium
playwright install-deps   # Linux のみ
```

- TOTP_SECRET の取得:
    - 2FA 初期設定画面で「シークレットキーを表示」リンクから base32 文字列（例: `JBSWY3DPEHPK3PXP`）
    - QR コードのみの場合 → スクショ撮影 → PC へ転送 → 以下で URL 抽出:
        - macOS: `brew install zbar` → `zbarimg --quiet --raw screenshot.png`
        - Linux: `sudo apt install zbar-tools` → `zbarimg --quiet --raw screenshot.png`
        - Windows: `pip install zbar-py` の Python パッケージ経由
    - 抽出された URL `otpauth://totp/...?secret=JBSWY3DPEHPK3PXP&algorithm=SHA1&digits=6&period=30` の各パラメータが TOTP のパラメータ
- 環境変数 `EMAIL` / `PASSWORD` / `TOTP_PROVISIONING_URI` を export（**コード直書き禁止**、secret manager 推奨）

## コード

```python
import os
import stat
import pyotp
from playwright.sync_api import sync_playwright

EMAIL = os.environ['EMAIL']
PASSWORD = os.environ['PASSWORD']
TOTP_URI = os.environ['TOTP_PROVISIONING_URI']   # otpauth://totp/...
HEADLESS = os.environ.get('HEADLESS', 'true').lower() == 'true'
STORAGE_STATE = 'state.json'

# pyotp.parse_uri は algorithm / digits / period を URI から読む
# (RFC 6238 §1.2: algorithm SHA1/SHA256/SHA512, digits 6/8, period 30/60 をサポート)
totp = pyotp.parse_uri(TOTP_URI)

with sync_playwright() as p:
    browser = p.chromium.launch(headless=HEADLESS)
    ctx = browser.new_context(
        storage_state=STORAGE_STATE if os.path.exists(STORAGE_STATE) else None,
    )
    page = ctx.new_page()

    # password / TOTP frame を screenshot / trace から除外する mask
    sensitive_locators = [
        page.locator('input[type=password]'),
        page.locator('input[name=totp]'),
    ]

    page.goto('https://app.example.com/dashboard')
    if 'login' in page.url:
        page.get_by_role('textbox', name='Email').fill(EMAIL)
        page.get_by_role('textbox', name='Password').fill(PASSWORD)
        page.get_by_role('button', name='Sign in').click()
        page.wait_for_selector('input[name=totp]')
        page.fill('input[name=totp]', totp.now())
        page.get_by_role('button', name='Verify').click()
        page.wait_for_url('**/dashboard', timeout=10_000)
        ctx.storage_state(path=STORAGE_STATE)
        os.chmod(STORAGE_STATE, stat.S_IRUSR | stat.S_IWUSR)  # 0600

    # screenshot は mask で機密 frame を塗りつぶす
    page.screenshot(path='after-login.png', mask=sensitive_locators)

    resp = page.request.get('https://app.example.com/api/orders')
    print('status:', resp.status)
    print('body:', resp.json())
    browser.close()
```

## tracing / video を取る場合の機密 frame 対応

CI で `trace.zip` / `video.mp4` を artifact upload するときは password / TOTP frame が混入する。

```python
# trace
ctx.tracing.start(snapshots=True, screenshots=True, sources=True)
# ... operations ...
ctx.tracing.stop(path='trace.zip')

# screenshot / trace の mask は同じ locator を使う
# (locator は run time で評価されるので mask 値は record 時点の DOM に対して効く)
page.screenshot(path='after-login.png', mask=sensitive_locators, mask_color='#FF0000')
```

[Playwright docs: Masking](https://playwright.dev/python/docs/screenshots#mask-locators) / [Playwright docs: Tracing](https://playwright.dev/python/docs/trace-viewer)。`trace.zip` / `videos/` は artifact retention を **1-3 日に短縮** + 暗号化保管。

## 期待出力

- 初回実行時に `state.json` が保存される（数 KB の JSON、認証 Cookie / localStorage 込み）
- 2 回目以降は `state.json` を読んでログインスキップ
- `after-login.png` にダッシュボード screenshot（password / TOTP frame が mask されている）
- stdout: `status: 200` + JSON body

## ハマりポイント

### artifact 漏洩経路 (Playwright で頻発)

| artifact | 機密の漏れ方 | 対策 |
|---|---|---|
| `screenshot()` | ログイン直後の dashboard に email / 通知バッジ | `mask=[locator]` を必ず渡す |
| `tracing.start(snapshots=True, screenshots=True)` | password 入力 step が `trace.zip` 内 screenshot に保存 | sensitive step の前後で `tracing.start_chunk()` / `stop_chunk()` を分けるか、`screenshots=False` |
| `record_video_dir` | TOTP 入力フレームが `mp4` 全体に残る | record_video は使わない / retention 短縮 / mask は video には効かない |
| `state.json` | session cookie + localStorage 全部 | `.gitignore` + `chmod 600` + secret manager 経由で fetch |

CI の artifact retention は 1-3 日に短縮し、public artifact にしない。

### TOTP の罠 (RFC 6238)

- **algorithm / digits / period は provisioning URI から読む**。`pyotp.TOTP(secret)` 直書きでは SHA1 / 6 桁 / 30 秒固定だが、サービスによっては SHA256 / 8 桁を使う
- TOTP は時計依存。CI host の時刻が NTP 同期されていないと `code mismatch` で fail → `chronyd` / `timedatectl status` で確認
- HOTP (counter ベース、[RFC 4226](https://datatracker.ietf.org/doc/html/rfc4226)) は別物。本事例の対象外
- seed の base32 は padding を含めないことが多い (Google Authenticator 仕様)

### state.json (storage_state) の運用

- **寿命管理**: 取得日時を JSON 外に持ち、24h 超で再ログイン
- **secret manager 経由**: AWS Secrets Manager / GCP Secret Manager に保存、起動時に fetch
- **device binding**: login 時の IP を記録、別 IP で使うときは再ログイン
- **account pool**: state-account_a.json / state-account_b.json で round-robin & rate sharing
- `.gitignore` + `chmod 600`

### Selector / 待機

- selector は `role` ベース推奨。`button[type=submit]` は壊れやすい
- `wait_until='networkidle'` は WebSocket / SSE がある SPA で never-true。`wait_for_selector` / `wait_for_url` で要素ベース待機

### CAPTCHA / その他

- CAPTCHA 出現時の判定 cheat sheet は [01/E-headless-stealth.md](../01-sns-dynamic-site/E-headless-stealth.md#captcha--waf-の判定-cheat-sheet) 参照。reCAPTCHA v2 image なら 2captcha / CapSolver で 15-30s で解ける
- デバッグ後に `HEADLESS=true` に戻し忘れると CI で画面が立ち上がろうとして失敗
- TOTP_SECRET / TOTP_URI は環境変数 / Vault / AWS Secrets Manager / GCP Secret Manager で管理。コード / git に直書き禁止
