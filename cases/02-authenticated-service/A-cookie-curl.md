# 事例 2-A: Cookie 抽出 + curl

## 前提 / install

- ブラウザで対象サービスにログイン（2FA 含む完全ログイン）
- Cookie 抽出と Netscape 形式の cookies.txt 生成手順は **[事例 1-B](../01-sns-dynamic-site/B-cookie-curl.md) と同じ**（EditThisCookie / Get cookies.txt LOCALLY 拡張機能、または DevTools → Application → Cookies の手動転写）
- `cookies.txt` は **必ず `.gitignore` に追加**

## コード

```bash
CSRF_TOKEN='<value>'
UA='Mozilla/5.0 ... Chrome/120.0 Safari/537.36'
curl -b cookies.txt -H "User-Agent: ${UA}" -H 'Accept: application/json' -H "X-CSRF-Token: ${CSRF_TOKEN}" -H 'X-Requested-With: XMLHttpRequest' 'https://internal.example.com/api/v1/orders' -o orders.json -w 'http_code=%{http_code} size=%{size_download}\n'
```

## 期待出力

- 標準出力: `http_code=200 size=42153`
- `orders.json` 構造例: `{"orders":[{"id":"ord_001","total":1500,"created_at":"2026-06-11T10:00:00Z"}],"total_count":123,"has_next":true}`
- `http_code=401 / 403`: Cookie 期限切れ → 再ログインして取り直し
- `http_code=302`: ログイン画面 redirect。`curl -L -v` で追跡

## ハマりポイント

- session の有効期限はサービスにより数時間〜数週間。長期運用するなら [B（Device Flow）](B-device-flow.md)か [C（Playwright）](C-playwright.md)が安定
- CSRF token が必須のエンドポイント: DevTools → Network → 対象 request → Headers タブで「X-」プレフィックス header と Form Data を全部確認
- `cookies.txt` が漏れると session 漏洩。`.gitignore` 必須
- 一般原則は [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
