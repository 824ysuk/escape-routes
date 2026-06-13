# 事例 2-E: ブラウザ拡張機能で抜く

> **対象範囲**: 拡張をインストールする PC のユーザーが対象サービスの正当な利用者本人であること。共用 PC / キオスク端末 / 他人の PC への install は不正アクセス禁止法 3 条・3 条の 2 (識別符号窃用 / 攻撃用プログラム) に該当しうる。会社支給端末では IT 部門の許可を取る。

## 前提 / install

- ローカル directory にファイルを 3 つ準備:
    - `my-extension/manifest.json`
    - `my-extension/background.js`
    - `my-extension/icon.png`（16/48/128 px、任意の透過 PNG）
- Chrome に install: `chrome://extensions/` → 右上「デベロッパーモード」ON → 「パッケージ化されていない拡張機能を読み込む」→ `my-extension/` を選択
    - **デベロッパーモードは Chrome Web Store の Limited Use 制約 / Manifest V3 review プロセスを完全に bypass する**。信頼境界が利用者本人に集中する点を理解した上で使う
- 受信側 collector（Cloud Functions / Cloudflare Workers / 自前 Express）を別途用意

## コード

manifest.json:

```json
{
  "manifest_version": 3,
  "name": "Order Logger",
  "version": "0.1",
  "description": "Log Order API responses to my collector",
  "permissions": ["webRequest", "storage", "alarms"],
  "host_permissions": ["https://app.example.com/*"],
  "background": { "service_worker": "background.js" },
  "icons": { "48": "icon.png" }
}
```

background.js:

```javascript
// URL の query string / fragment に access_token / session id が乗ることが
// 実在する API (OAuth implicit, redirect 中継, 古い CSRF token を URL に付与する
// 設計 等)。collector に送る前に sensitive な key を *** に置換し fragment は捨てる。
const SENSITIVE_QUERY_KEYS = ['access_token', 'token', 'session', 'sid', 'auth', 'apikey', 'api_key', 'code', 'state'];
function maskUrl(raw) {
  const u = new URL(raw);
  for (const k of [...u.searchParams.keys()]) {
    if (SENSITIVE_QUERY_KEYS.includes(k.toLowerCase())) u.searchParams.set(k, '***');
  }
  u.hash = '';
  return u.toString();
}

// collector への認証 (shared secret)。chrome.storage に手動で投入。
async function collectorSecret() {
  const { collectorSecret } = await chrome.storage.local.get('collectorSecret');
  return collectorSecret;
}

chrome.webRequest.onCompleted.addListener(
  async (details) => {
    if (details.method !== 'GET' || details.statusCode !== 200) return;
    const secret = await collectorSecret();
    if (!secret) { console.warn('collectorSecret 未設定'); return; }
    try {
      await fetch('https://collector.example.com/log', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Collector-Auth': secret,
        },
        body: JSON.stringify({ url: maskUrl(details.url), ts: Date.now() }),
      });
    } catch (e) {
      console.error('collector POST failed', e);
    }
  },
  { urls: ['https://app.example.com/api/orders*'] }
);

chrome.alarms.create('keep-alive', { periodInMinutes: 1 });
chrome.alarms.onAlarm.addListener(() => console.log('alive'));
```

受信側 collector の最小例（Cloudflare Workers）:

```javascript
export default {
  async fetch(request, env) {
    if (request.method !== 'POST') return new Response('Method Not Allowed', { status: 405 });
    if (request.headers.get('X-Collector-Auth') !== env.COLLECTOR_SECRET) {
      return new Response('forbidden', { status: 403 });
    }
    const body = await request.json();
    console.log('received:', body);
    return new Response('ok');
  }
};
```

collector の Worker に環境変数 `COLLECTOR_SECRET` を設定（`wrangler secret put COLLECTOR_SECRET`）し、拡張側にも同じ値を投入する:

```javascript
// 拡張のサービスワーカー console から 1 回だけ実行
chrome.storage.local.set({ collectorSecret: '<長いランダム文字列>' });
```

`webRequest` は header と URL のみ取得可能で body は取れない。response body が必要なら content script で `fetch` を wrap する:

```javascript
// content_script.js — 対象ページに inject (manifest.json の content_scripts で指定)
const origFetch = window.fetch;
const SENSITIVE_KEYS = /^(password|token|secret|email|phone|ssn|card)/i;
function redact(obj) {
  if (Array.isArray(obj)) return obj.map(redact);
  if (obj && typeof obj === 'object') {
    return Object.fromEntries(
      Object.entries(obj).map(([k, v]) => [k, SENSITIVE_KEYS.test(k) ? '***' : redact(v)])
    );
  }
  return obj;
}
window.fetch = new Proxy(origFetch, {
  apply: async (target, thisArg, args) => {
    const resp = await Reflect.apply(target, thisArg, args);
    const url = typeof args[0] === 'string' ? args[0] : args[0].url;
    if (url.includes('/api/orders')) {
      const clone = resp.clone();
      clone.json().then((body) => {
        chrome.runtime.sendMessage({ type: 'order', body: redact(body) });
      });
    }
    return resp;
  },
});
```

## 期待出力

- `chrome://extensions/` で対象拡張が「有効」と表示
- ユーザーが対象サイトを操作 → collector のログに `received: {url: 'https://app.example.com/api/orders?...', ts: 1718000000000}` が出る
- 拡張のデバッグ: `chrome://extensions/` → 拡張のカード → 「サービスワーカー」リンク → DevTools で console / network 観察

## 配布方法の選択肢

本サンプルは sideload（デベロッパーモード）前提。組織・公開目的に応じて以下の経路がある。

| 配布方法 | 対象 | Chrome Web Store 審査 | 利用者操作 | 主な用途 |
|---|---|---|---|---|
| sideload (デベロッパーモード) | 開発者本人のみ | なし | 手動で読み込み | 個人実験 / 開発中 |
| Listed (Public) | 不特定多数 | あり (Limited Use 準拠) | Store からインストール | 一般公開 |
| Unlisted | URL を知っている人 | あり (Limited Use 準拠) | Store URL からインストール | 社内配布 / 限定公開 |
| Enterprise Policy (Force Install) | Chrome Enterprise 配下端末 | なし (組織管理者の責任) | 端末側で自動インストール | 業務端末向け強制配布 |

Listed / Unlisted への移行時には manifest の `key` 固定、Limited Use Policy 同意が追加で必要。Enterprise Policy への移行時は組織 OU 設定が必要。詳細は公式 docs を参照:

- [Chrome Enterprise: Force install extensions](https://support.google.com/chrome/a/answer/9296680)
- [Chrome Web Store: Publishing guidelines](https://developer.chrome.com/docs/webstore/publish/)

## ハマりポイント

- Manifest V3 の service worker は 30 秒の idle で停止 → 上の `chrome.alarms` で keep-alive
- `webRequest` API では HTTP header と URL のみ取得可能、レスポンス body は取得不能 → body が必要なら content_scripts で `fetch` を wrap
- Chrome Web Store 公開時は permission 説明文の justification が必要
- **`webRequest` は Manifest V3 で観測用途なら [`declarativeNetRequest`](https://developer.chrome.com/docs/extensions/reference/api/declarativeNetRequest) が推奨**。`webRequest` の blocking 形態は enterprise policy 限定になりつつあり、Web Store 公開の際の justification 要件が厳しい。`permissions: ['webRequest']` は「閲覧履歴を読み取れる」と表示されるため、エンドユーザーへの説明責任とプライバシー影響評価（PIA）を社内で行うか、[Chrome Enterprise policy 配布](https://developer.chrome.com/docs/webstore/program-policies/permissions/)に切り替えるかを検討
- **`host_permissions` を `<all_urls>` に拡げない**（Web Store reject 直行）。常に必要最小のホストに絞る
- collector へ送る URL に PII / token が混入するため、上のコードでは (a) 拡張側で sensitive query を `***` に置換、(b) collector を `X-Collector-Auth` shared secret で認証、(c) 拡張・collector を同 TLS 経路に限定する形にしている。これでも不安なら、URL ではなく [WebRequest API のレスポンス header から必要 metadata だけ抽出して送る](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html#sensitive-information-in-http-requests)、Logging Cheat Sheet 準拠の masking を追加する
