# 事例 2-E: ブラウザ拡張機能で抜く

## 前提 / install

- ローカル directory にファイルを 3 つ準備:
    - `my-extension/manifest.json`
    - `my-extension/background.js`
    - `my-extension/icon.png`（16/48/128 px、任意の透過 PNG）
- Chrome に install: `chrome://extensions/` → 右上「デベロッパーモード」ON → 「パッケージ化されていない拡張機能を読み込む」→ `my-extension/` を選択
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
chrome.webRequest.onCompleted.addListener(
  async (details) => {
    if (details.method === 'GET' && details.statusCode === 200) {
      try {
        await fetch('https://collector.example.com/log', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ url: details.url, ts: Date.now() }),
        });
      } catch (e) {
        console.error('collector POST failed', e);
      }
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
  async fetch(request) {
    if (request.method !== 'POST') return new Response('Method Not Allowed', { status: 405 });
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

- Manifest V3 の service worker は 30 秒の idle で停止 → 上の `chrome.alarms` で keep-alive
- `webRequest` API では HTTP header と URL のみ取得可能、レスポンス body は取得不能 → body が必要なら content_scripts で `fetch` を wrap
- Chrome Web Store 公開時は permission 説明文の justification が必要
- collector へ送る URL の query string に session token / PII が含まれることがある。送信前にマスクするか、collector 側を認証付き・アクセス制限ありにする
