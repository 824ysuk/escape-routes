# 事例 5-E: アプリにデバッグログを仕込む

## 前提 / install

install 不要、対象アプリのソースが手元にある前提

## コード

言語別、すべて Authorization のマスク実装込み。

Python (requests):

```python
import http.client as http_client
import logging
import requests

http_client.HTTPConnection.debuglevel = 1
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s %(name)s %(levelname)s %(message)s')
logging.getLogger('urllib3').setLevel(logging.DEBUG)

def mask_headers(headers):
    return {k: ('***MASKED***' if k.lower() in ('authorization', 'cookie', 'x-api-key') else v) for k, v in headers.items()}

def safe_get(url, **kw):
    headers = kw.get('headers', {})
    logging.info(f'GET {url} headers={mask_headers(headers)}')
    return requests.get(url, **kw)

r = safe_get('https://api.example.com/orders', headers={'Authorization': 'Bearer abc123'})
print(r.status_code, r.text[:200])
```

Node.js (組み込み http) — masking 込み:

```javascript
// $ node script.js
import https from 'node:https';

const SECRET_HEADERS = ['authorization', 'cookie', 'x-api-key'];
function maskHeaders(headers) {
  return Object.fromEntries(
    Object.entries(headers).map(([k, v]) =>
      [k, SECRET_HEADERS.includes(k.toLowerCase()) ? '***MASKED***' : v]
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

Go (net/http/httptrace) — Authorization 除外:

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptrace"
	"strings"
)

func main() {
	secretHeaders := map[string]bool{"authorization": true, "cookie": true, "x-api-key": true}
	trace := &httptrace.ClientTrace{
		DNSDone: func(d httptrace.DNSDoneInfo) { fmt.Printf("DNS: %+v\n", d) },
		GotConn: func(c httptrace.GotConnInfo) { fmt.Printf("GotConn: %+v\n", c) },
		WroteHeaderField: func(k string, v []string) {
			if secretHeaders[strings.ToLower(k)] {
				fmt.Printf("hdr %s=***MASKED***\n", k)
			} else {
				fmt.Printf("hdr %s=%v\n", k, v)
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

- **Authorization / Cookie / X-API-Key を必ずマスク**: 全 3 言語のサンプルに mask 関数を入れた。masking なしで本番ログに流すと token 漏洩
- production には debug ログを残さない。build flag / env var で切り替える（`if process.env.DEBUG`）
- Python `http.client.HTTPConnection.debuglevel = 1` は send のみ詳細、response body は別途 `r.text` を log
