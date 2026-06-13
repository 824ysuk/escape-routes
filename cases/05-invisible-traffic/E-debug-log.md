# 事例 5-E: アプリにデバッグログを仕込む

> **対象範囲**: 自社が開発・運用するアプリに限る (ソースが手元にある前提)。

## 前提 / install

install 不要、対象アプリのソースが手元にある前提

## マスキング方針: allow-list を default に

固定 deny-list (`Authorization` / `Cookie` / `X-API-Key`) は新 auth scheme を取りこぼす。modern API は以下のような credential 性 header を多数持つ:

| Header | 出典 |
|---|---|
| `Authorization` / `Proxy-Authorization` | RFC 7235 |
| `Cookie` / `Set-Cookie` | RFC 6265 |
| `X-API-Key` / `x-amz-security-token` / `x-goog-api-key` / `x-ms-client-secret` | クラウドベンダー固有 |
| `X-CSRF-Token` / `X-Anti-Forgery-Token` | アプリ固有 |
| `DPoP` | [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449) sender-constrained Bearer |
| `Signature` / `Signature-Input` | [RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421) HTTP Message Signatures |

**推奨**: **allow-list (既知の safe header だけ通し、未知は `***MASKED***`)**。サンプル実装は下記コードに含まれる。

## コード

言語別、すべて allow-list マスク実装込み。

### Python (requests)

```python
import http.client as http_client
import logging
import re
import requests

http_client.HTTPConnection.debuglevel = 1
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s %(name)s %(levelname)s %(message)s')
logging.getLogger('urllib3').setLevel(logging.DEBUG)

# allow-list: 表示してよい header のみ通す
SAFE_HEADERS = {
    'accept', 'accept-encoding', 'accept-language',
    'content-type', 'content-length',
    'host', 'user-agent', 'connection',
    'cache-control', 'pragma', 'expires',
    'date', 'server', 'via', 'x-request-id', 'x-correlation-id',
}

# 残存 secret 検出 (JWT, base64-ish, hex token)
SECRET_LIKE = re.compile(r'(eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}|[A-Fa-f0-9]{32,})')

def mask_headers(headers):
    out = {}
    for k, v in headers.items():
        if k.lower() in SAFE_HEADERS:
            # 値に secret-like が混じっていないか確認
            if isinstance(v, str) and SECRET_LIKE.search(v):
                out[k] = '***SECRET-LIKE***'
            else:
                out[k] = v
        else:
            out[k] = '***MASKED***'
    return out

def safe_get(url, **kw):
    headers = kw.get('headers', {})
    logging.info(f'GET {url} headers={mask_headers(headers)}')
    return requests.get(url, **kw)

r = safe_get('https://api.example.com/orders', headers={'Authorization': 'Bearer abc123'})
print(r.status_code, r.text[:200])
```

### Node.js (組み込み http) — allow-list 込み

```javascript
// $ node script.js
import https from 'node:https';

const SAFE_HEADERS = new Set([
  'accept', 'accept-encoding', 'accept-language',
  'content-type', 'content-length',
  'host', 'user-agent', 'connection',
  'cache-control', 'pragma', 'expires',
  'date', 'server', 'via', 'x-request-id', 'x-correlation-id',
]);

function maskHeaders(headers) {
  return Object.fromEntries(
    Object.entries(headers).map(([k, v]) =>
      [k, SAFE_HEADERS.has(k.toLowerCase()) ? v : '***MASKED***']
    )
  );
}

function safeGet(url, options) {
  console.error(`GET ${url} headers=${JSON.stringify(maskHeaders(options.headers || {}))}`);
  return new Promise((resolve, reject) => {
    const req = https.get(url, options, res => {
      let body = '';
      res.on('data', chunk => body += chunk);
      res.on('end', () => {
        console.error(`status=${res.statusCode} body=${body.slice(0, 200)}`);
        resolve({ status: res.statusCode, body });
      });
    });
    req.on('error', reject);
  });
}

await safeGet('https://api.example.com/orders', { headers: { 'Authorization': 'Bearer abc123' } });
```

### Go (net/http/httptrace) — allow-list

注: `WroteHeaderField` は **ヘッダがソケットに書き込まれた後** に呼ばれる notification callback ([net/http/httptrace docs](https://pkg.go.dev/net/http/httptrace#ClientTrace.WroteHeaderField))。ここでのマスクは「ログ表示のマスク」のみで、実通信には real な `Authorization: Bearer abc123` がそのまま送出される。Python / Node 版の `safe_get` / `safeGet` は dict から消すので送信からも除外される。送信から消したい場合は `req.Header.Del("Authorization")` 等で別途処理する。

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptrace"
	"strings"
)

func main() {
	safeHeaders := map[string]bool{
		"accept": true, "accept-encoding": true, "accept-language": true,
		"content-type": true, "content-length": true,
		"host": true, "user-agent": true, "connection": true,
		"cache-control": true, "pragma": true, "expires": true,
		"date": true, "server": true, "via": true,
		"x-request-id": true, "x-correlation-id": true,
	}
	trace := &httptrace.ClientTrace{
		DNSDone: func(d httptrace.DNSDoneInfo) { fmt.Printf("DNS: %+v\n", d) },
		GotConn: func(c httptrace.GotConnInfo) { fmt.Printf("GotConn: %+v\n", c) },
		WroteHeaderField: func(k string, v []string) {
			if safeHeaders[strings.ToLower(k)] {
				fmt.Printf("hdr %s=%v\n", k, v)
			} else {
				fmt.Printf("hdr %s=***MASKED***\n", k)
			}
		},
		GotFirstResponseByte: func() { fmt.Println("got first byte") },
	}
	req, _ := http.NewRequest("GET", "https://api.example.com/orders", nil)
	req.Header.Set("Authorization", "Bearer abc123")
	req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
	resp, _ := http.DefaultTransport.RoundTrip(req)
	fmt.Println("status:", resp.StatusCode)
}
```

## 期待出力

- Python: stderr に `2026-06-11 ... INFO GET https://api.example.com/orders headers={'Authorization': '***MASKED***'}` + 続いて urllib3 の DEBUG ログ
- Node.js: stderr に `GET https://... headers={"authorization":"***MASKED***"}` + `status=200 body={...}`
- Go: stdout に `hdr Authorization=***MASKED***` `DNS: {...}` `GotConn: {...}` `status: 200`

## ハマりポイント

### allow-list vs deny-list

- **allow-list を default にする**。deny-list は新 auth header (DPoP, HTTP Message Signatures, クラウドベンダー固有) を取りこぼす
- 残存 secret 検出 (regex で JWT-like / base64-ish) を 2 段目に置くと安全網になる (Python 例参照)

### Production への deploy

- production には debug ログを残さない。build flag / env var で切り替える (`if process.env.DEBUG`)
- log shipper の collector 側でも 2 段目マスクを設定 (defense-in-depth)
- 新規 auth scheme (DPoP / HTTP Message Signatures / mTLS Token Binding) を導入した際は allow-list / deny-list を見直す

### JWT bearer の部分開示 (発展)

JWT bearer は `header.payload.signature` の 3 部構成。`header` 部 (alg, kid, typ) は signing 設定でありデバッグに有用。全マスクではなく `<jwt header>.***.***` のような部分開示でデバッグ精度が上がる場合がある (例: `alg=none` 攻撃の検知、kid rotation 確認)。

- [RFC 7519 (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 7515 (JWS)](https://datatracker.ietf.org/doc/html/rfc7515)

### その他

- Python `http.client.HTTPConnection.debuglevel = 1` は send のみ詳細、response body は別途 `r.text` を log
- [OWASP Logging Cheat Sheet - Data to Exclude](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html#data-to-exclude) 参照
