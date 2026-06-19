# 事例 2-F: WebAuthn Virtual Authenticator

> **対象範囲**: 対象は利用者自身のアカウント、または自社が運用するサービスに限る。テスト目的の自動化に限定 — 本番認証 bypass・他人アカウントへの悪用は不正アクセス禁止法 3 条に該当しうる。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

事例 2-C は通常 2FA の TOTP を Playwright で自動入力するが、**WebAuthn / passkey-only サイト**には無力（pyotp は TOTP のみ。WebAuthn ceremony で要求される user presence / user verification を headless E2E では通せない）。本手段は [W3C WebAuthn L3 §11](https://www.w3.org/TR/webauthn-3/) の WebDriver Extension を使い、authenticator 側を仮想化することで ceremony 全体を自動化する。

## 前提 / install

### Selenium Python

```bash
pip install selenium
# Chrome 対応 WebDriver が PATH に存在することを確認 (ChromeDriver)
python -c "from selenium.webdriver.common.virtual_authenticator import VirtualAuthenticatorOptions; print('ok')"
```

- `VirtualAuthenticatorOptions` は `selenium >= 4.0` (2021-10) から公式サポート ([Selenium docs](https://www.selenium.dev/documentation/webdriver/interactions/virtual_authenticator/))
- Chrome / Edge で動作確認。Firefox は CDP WebAuthn domain 非対応のため Selenium binding も不可

### Playwright Node（CDP WebAuthn domain）

```bash
npm install playwright
npx playwright install chromium
```

- `WebAuthn.enable` / `addVirtualAuthenticator` は Chrome DevTools Protocol (CDP) の機能のため **Chromium 系のみ**対応
- Puppeteer も `CDPSession` 経由で同じ命令を送れる（API は同一）

### 対象サイトの確認事項

使用前に対象サービスの WebAuthn 設定を確認する:

| 確認項目 | 確認方法 | Virtual Authenticator への影響 |
|---|---|---|
| `protocol`: ctap2 / u2f | RP の `pubKeyCredParams` algorithm を DevTools Network で確認 | `protocol` を合わせる (mismatch → NotAllowedError) |
| UV 強制 (`userVerification: "required"`)  | `navigator.credentials.create()` / `get()` の options | `hasUserVerification: true` + `isUserVerified: true` の双立て必須 |
| Discoverable credential 要求（usernameless flow） | Conditional UI 等で `allowCredentials` が空の `get()` | `hasResidentKey: true` + `isResidentCredential: true` で inject 必須 |

## コード

### Selenium Python（公式 binding）

```python
from selenium import webdriver
from selenium.webdriver.common.virtual_authenticator import (
    Credential,
    VirtualAuthenticatorOptions,
)

driver = webdriver.Chrome()

# 1. Virtual authenticator を登録
options = VirtualAuthenticatorOptions()
options.protocol = VirtualAuthenticatorOptions.Protocol.CTAP2
options.transport = VirtualAuthenticatorOptions.Transport.INTERNAL
options.has_resident_key = True       # discoverable credential を許可
options.has_user_verification = True  # UV capability あり
options.is_user_verified = True       # UV ceremony を常に成功扱い
driver.add_virtual_authenticator(options)
print("[DEBUG] virtual authenticator added")

# 2. (任意) credential を直接 inject — 登録 ceremony をスキップする場合
credential = Credential.create_non_resident_credential(
    credential_id=b"\x01\x02\x03\x04",
    rp_id="example.com",
    private_key=b"...PKCS#8 bytes...",
    sign_count=0,
)
driver.add_credential(credential)
print("[DEBUG] credential injected: id=01020304, rp_id=example.com")

# 3. テスト本体 — navigator.credentials.get() / create() が virtual authenticator で完結
driver.get("https://example.com/login")

# 4. 登録された credential を確認
creds = driver.get_credentials()
print(f"[DEBUG] get_credentials() -> {len(creds)} credential(s)")

# 5. cleanup
driver.remove_all_credentials()
driver.remove_virtual_authenticator()
driver.quit()
```

### Playwright Node（CDP WebAuthn domain）

```javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();

  // 1. CDP session を取得して WebAuthn domain を有効化
  //    WebAuthn.enable を呼ぶ前に addVirtualAuthenticator を発行すると
  //    "Error: WebAuthn is not enabled" で失敗する。順序に注意。
  const client = await context.newCDPSession(page);
  await client.send('WebAuthn.enable');

  // 2. Virtual authenticator を登録 (authenticatorId が返る)
  const { authenticatorId } = await client.send(
    'WebAuthn.addVirtualAuthenticator',
    {
      options: {
        protocol: 'ctap2',
        transport: 'internal',
        hasResidentKey: true,
        hasUserVerification: true,
        isUserVerified: true,
        automaticPresenceSimulation: true,   // false にすると get() が hang する
      },
    }
  );

  // 3. (任意) credential を直接 inject
  await client.send('WebAuthn.addCredential', {
    authenticatorId,
    credential: {
      credentialId: Buffer.from([1, 2, 3, 4]).toString('base64'),
      isResidentCredential: true,
      rpId: 'example.com',
      privateKey: '...PKCS#8 base64...',    // raw DER / PEM / JWK は Invalid private key
      userHandle: Buffer.from('user-123').toString('base64'),
      signCount: 0,
    },
  });

  // 4. UV を後から切替
  await client.send('WebAuthn.setUserVerified', {
    authenticatorId,
    isUserVerified: true,
  });

  await page.goto('https://example.com/login');

  // 5. 登録された credential を確認
  const { credentials } = await client.send('WebAuthn.getCredentials', {
    authenticatorId,
  });
  console.log('credentials:', credentials.length);

  await browser.close();
})();
```

## 期待出力

### Selenium Python (`add_virtual_authenticator` → `get_credentials` 経由)

```
[DEBUG] virtual authenticator added
[DEBUG] credential injected: id=01020304, rp_id=example.com
[DEBUG] get_credentials() -> 1 credential(s)
```

### Playwright Node (CDP)

```
authenticatorId: 802ec27f-3c7d-4d2a-9b8f-1a2b3c4d5e6f
credentials: 1
```

## ハマりポイント

### protocol mismatch（CTAP2 / U2F と RP の要求 algorithm のズレ）

RP が `pubKeyCredParams` で ES256（CTAP2）のみを要求しているのに `protocol: 'u2f'` で authenticator を作成すると `navigator.credentials.create()` が `NotAllowedError` を返す。DevTools → Network → `navigator.credentials` 呼び出し時の request body で `pubKeyCredParams` を確認し、RP の要求 algorithm と `protocol`（`ctap2` or `u2f`）を合わせる。

### UV-required ceremony での `hasUserVerification` / `isUserVerified` 双立て

- `hasUserVerification: true` — authenticator が UV capability を持つことを宣言（capability フラグ）
- `isUserVerified: true` — 今回の ceremony で UV が成功したとして扱う（ceremony 結果フラグ）

`hasUserVerification: true` だけで `isUserVerified: false` のままにすると、UV を要求する assertion で `NotAllowedError` が返る。2 つを同時に立てる。

### Discoverable credential と Conditional UI (usernameless flow)

`allowCredentials` が空の `navigator.credentials.get()`（passkey autofill / usernameless login）では、authenticator に resident credential が存在しないと credential が見つからない。

```python
# non-resident ではなく resident credential で inject する
credential = Credential.create_resident_credential(
    credential_id=b"\x01\x02\x03\x04",
    rp_id="example.com",
    user_handle=b"user-123",
    private_key=b"...PKCS#8 bytes...",
    sign_count=0,
)
```

CDP 側も `isResidentCredential: true` が必要。non-resident で inject すると allow-list なしの `get()` で credential が見つからない。

### Conditional UI (passkey autofill) の feature detection

passkey autofill を使うサービスでは `isConditionalMediationAvailable()` を feature detect する実装が多い。Chromium でも Virtual Authenticator 環境では挙動が変わりうるため、`typeof` ガード付きで確認する:

```javascript
if (
  typeof PublicKeyCredential !== 'undefined' &&
  typeof PublicKeyCredential.isConditionalMediationAvailable === 'function'
) {
  const available = await PublicKeyCredential.isConditionalMediationAvailable();
  // available === false の場合は modal flow に fallback
}
```

未対応ブラウザでは `TypeError` になるため `typeof` ガードは必須。

### Playwright CDP の `WebAuthn.enable` 順序と `automaticPresenceSimulation`

- **`WebAuthn.enable` を先に呼ぶ**: `addVirtualAuthenticator` を先に発行すると `Error: WebAuthn is not enabled` で失敗する。CDP session 取得直後に `WebAuthn.enable` を 1 回送ること。
- **`automaticPresenceSimulation`**: `false` にしたまま `setAutomaticPresenceSimulation` で再有効化を忘れると user presence test が解決されず `navigator.credentials.get()` が hang する（CDP 仕様: `false` 時は手動 trigger 必須）。
- **`privateKey` は PKCS#8 を base64 で渡す**: raw DER / PEM / JWK を渡すと `Invalid private key` で `addCredential` が失敗する。

### 事例 2-C との使い分け

| | 事例 2-C (Playwright + pyotp) | 事例 2-F (Virtual Authenticator) |
|---|---|---|
| 認証方式 | TOTP (RFC 6238 / 4226) | WebAuthn / passkey |
| 詰まる点 | WebAuthn ceremony は通せない | TOTP 自動入力は不要 |
| 典型サービス | TOTP 2FA 付き OAuth / Cookie 認証 | passkey-only / WebAuthn 必須サービス |
| 実ブラウザ操作 | 必要（UI fill） | 不要（authenticator 仮想化）|

## 参考

- [W3C WebAuthn Level 3 §11 User Agent Automation](https://www.w3.org/TR/webauthn-3/)
- [Selenium: Virtual Authenticator](https://www.selenium.dev/documentation/webdriver/interactions/virtual_authenticator/)
- [Chrome DevTools Protocol: WebAuthn domain](https://chromedevtools.github.io/devtools-protocol/tot/WebAuthn/)
- [Corbado: Passkeys E2E Testing with WebAuthn Virtual Authenticator](https://www.corbado.com/blog/passkeys-e2e-playwright-testing-webauthn-virtual-authenticator)
- [Yubico: Passkey Autofill / Conditional UI Implementation Guidance](https://developers.yubico.com/WebAuthn/Concepts/Passkey_Autofill/Implementation_Guidance/Autofill_-_Conditional_UI_Browser_Feature_Detection.html)
- [web.dev: WebAuthn Feature Detection](https://web.dev/articles/webauthn-feature-detection)
