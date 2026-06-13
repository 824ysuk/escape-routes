# 事例 2-C: Playwright で完全自動化

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
    - 抽出された URL `otpauth://totp/...?secret=JBSWY3DPEHPK3PXP&...` の `secret=` パラメータが TOTP_SECRET
- 環境変数 `EMAIL` / `PASSWORD` / `TOTP_SECRET` を export（**コード直書き禁止**、secret manager 推奨）

## コード

```python
import os
import stat
import pyotp
from playwright.sync_api import sync_playwright

EMAIL = os.environ['EMAIL']
PASSWORD = os.environ['PASSWORD']
TOTP_SECRET = os.environ['TOTP_SECRET']
HEADLESS = os.environ.get('HEADLESS', 'true').lower() == 'true'
STORAGE_STATE = 'state.json'

with sync_playwright() as p:
    browser = p.chromium.launch(headless=HEADLESS)
    ctx = browser.new_context(storage_state=STORAGE_STATE if os.path.exists(STORAGE_STATE) else None)
    page = ctx.new_page()
    page.goto('https://app.example.com/dashboard')
    if 'login' in page.url:
        page.get_by_role('textbox', name='Email').fill(EMAIL)
        page.get_by_role('textbox', name='Password').fill(PASSWORD)
        page.get_by_role('button', name='Sign in').click()
        page.wait_for_selector('input[name=totp]')
        page.fill('input[name=totp]', pyotp.TOTP(TOTP_SECRET).now())
        page.get_by_role('button', name='Verify').click()
        page.wait_for_url('**/dashboard', timeout=10_000)
        ctx.storage_state(path=STORAGE_STATE)
        os.chmod(STORAGE_STATE, stat.S_IRUSR | stat.S_IWUSR)  # 0600
    page.screenshot(path='after-login.png')
    resp = page.request.get('https://app.example.com/api/orders')
    print('status:', resp.status)
    print('body:', resp.json())
    browser.close()
```

## 期待出力

- 初回実行時に `state.json` が保存される（数 KB の JSON、認証 Cookie / localStorage 込み）
- 2 回目以降は `state.json` を読んでログインスキップ
- `after-login.png` にダッシュボード screenshot（数百 KB）
- stdout: `status: 200` + JSON body

## ハマりポイント

- selector は `role` ベース推奨。`button[type=submit]` は壊れやすい
- CAPTCHA 出現時は突破不能。開発時 `HEADLESS=false` で動作確認、production は別 IP / 別アカウント
- TOTP_SECRET は環境変数 / Vault / AWS Secrets Manager / GCP Secret Manager で管理。コード / git に直書き禁止。環境変数で渡しても `/proc/<pid>/environ` から同 uid プロセスに読まれるため、長期運用なら keyring / Vault に直接アクセスして取得する方が望ましい
- デバッグ後に `HEADLESS=true` に戻し忘れると CI で画面が立ち上がろうとして失敗
- `state.json` は session Cookie / localStorage / sessionStorage を平文 JSON で含む（完全な session hijack に直結）。`.gitignore` 追加 + `chmod 600`（上のコードに実装済み）+ 漏洩時は対象サービスで全 session ログアウト（[Playwright auth docs](https://playwright.dev/python/docs/auth#reuse-signed-in-state) / [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)）
