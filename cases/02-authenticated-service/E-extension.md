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
// MV3: webRequest は observe-only (request/response の body は取れない)
chrome.webRequest.onCompleted.addListener(
  async (details) => {
    if (details.method === 'GET' && details.statusCode === 200) {
      try {
        await fetch('https://collector.example.com/log', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer ' + COLLECTOR_TOKEN,   // collector を認証付きに
          },
          body: JSON.stringify({
            url: maskQuery(details.url),   // query string の token / PII をマスク
            ts: Date.now(),
          }),
        });
      } catch (e) {
        console.error('collector POST failed', e);
      }
    }
  },
  { urls: ['https://app.example.com/api/orders*'] }
);

function maskQuery(url) {
  const u = new URL(url);
  for (const key of [...u.searchParams.keys()]) {
    if (/token|secret|password|email|phone/i.test(key)) {
      u.searchParams.set(key, '***');
    }
  }
  return u.toString();
}

chrome.alarms.create('keep-alive', { periodInMinutes: 1 });
chrome.alarms.onAlarm.addListener(() => console.log('alive'));
```

response body が必要な場合 (content script で `fetch` を wrap):

```javascript
// content_script.js — 対象ページに inject
const origFetch = window.fetch;
window.fetch = new Proxy(origFetch, {
  apply: async (target, thisArg, args) => {
    const resp = await Reflect.apply(target, thisArg, args);
    const url = typeof args[0] === 'string' ? args[0] : args[0].url;
    if (url.includes('/api/orders')) {
      const clone = resp.clone();
      clone.json().then((body) => {
        // body をマスクしてから collector へ送る
        chrome.runtime.sendMessage({ type: 'order', body: redact(body) });
      });
    }
    return resp;
  },
});
```

受信側 collector の最小例 (Cloudflare Workers、認証付き):

```javascript
export default {
  async fetch(request, env) {
    if (request.method !== 'POST') return new Response('Method Not Allowed', { status: 405 });
    const auth = request.headers.get('Authorization');
    if (auth !== `Bearer ${env.COLLECTOR_TOKEN}`) {
      return new Response('Unauthorized', { status: 401 });
    }
    const body = await request.json();
    console.log('received:', body);
    return new Response('ok');
  }
};
```

## 期待出力

- `chrome://extensions/` で対象拡張が「有効」と表示
- ユーザーが対象サイトを操作 → collector のログに `received: {url: 'https://app.example.com/api/orders?...', ts: 1718000000000}` が出る
- 拡張のデバッグ: `chrome://extensions/` → 拡張のカード → 「サービスワーカー」リンク → DevTools で console / network 観察

## ハマりポイント

### Manifest V3 の制約 (2024+ 必須)

- **`webRequest` は observe-only**。response body は取れない (MV2 の `webRequestBlocking` は廃止)。body が必要なら content script で `fetch` / `XMLHttpRequest` を Proxy で wrap する
- `declarativeNetRequest` への移行が推奨されているが、observe (記録) には依然 `webRequest` を使う
- service worker は 30 秒の idle で停止 → `chrome.alarms` で keep-alive
- Chrome Web Store 公開時は `webRequest` permission の justification が厳しい。`optional_host_permissions` で runtime permission に分離するのが現代的

### 配布方法

| 方法 | 範囲 | 適用場面 |
|---|---|---|
| Web Store (Public) | 一般公開 | 一般向けプロダクト (Limited Use 制約あり) |
| Web Store (Unlisted) | URL を知る人のみ | 社内・限定配布 |
| Enterprise Policy (`ExtensionInstallForcelist`) | 組織ドメイン | 業務利用、IT 部門経由 |
| **デベロッパーモード sideload** | 開発者本人のみ | 個人実験 (Web Store 審査 bypass、信頼境界注意) |

長期運用するなら Unlisted か Enterprise Policy で配布する。

### Token / Cookie の漏洩経路

- collector へ送る URL の query string に session token / PII が含まれることがある → `maskQuery` でマスク
- collector エンドポイントは **認証付き** (Bearer token / mTLS) + アクセス制限 (IP allow-list / Cloudflare Access)
- collector の retention は 短期 (7-30 日) + 暗号化保管 + GDPR / 個情法に応じた DPA を結ぶ

### EditThisCookie 系拡張への注意

`EditThisCookie` は 2024 年に売却・malware 化したため、Cookie 抽出は [Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc) を推奨 ([01-B-cookie-curl.md](../01-sns-dynamic-site/B-cookie-curl.md) 参照)。Cookie 抽出系拡張は **最終確認日 / 開発者の同一性** を毎回確認する。

### 参考

- [Chrome Developers: Manifest V3 migration](https://developer.chrome.com/docs/extensions/develop/migrate)
- [Chrome Web Store: Limited Use Policy](https://developer.chrome.com/docs/webstore/program-policies/limited-use/)
- [Chrome Enterprise: ExtensionInstallForcelist](https://chromeenterprise.google/policies/#ExtensionInstallForcelist)
