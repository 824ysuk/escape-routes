# 事例 1-B: Cookie + curl + UA 偽装

## 前提 / install

- ブラウザで対象サイトにログイン
- Cookie 抽出は拡張機能を使うのが速い:
    - Chrome: [EditThisCookie](https://chromewebstore.google.com/detail/editthiscookie/fngmhnnpilhplaeedifhccceomclgfbg) / [Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc)
    - Firefox: [cookies.txt](https://addons.mozilla.org/firefox/addon/cookies-txt/)
- 手動の場合: DevTools → Application → Cookies → 全 cookie をテキストに転写
- curl install: macOS / Linux はプリインストール、Windows は `winget install curl.curl`
- `cookies.txt` は **必ず `.gitignore` に追加**（session 情報のため共有禁止）

Netscape 形式の `cookies.txt`（TAB 区切り 7 フィールド = domain / include subdomain (TRUE/FALSE) / path / secure (TRUE/FALSE) / expiration (UNIX timestamp) / name / value）:

```
# Netscape HTTP Cookie File
.instagram.com	TRUE	/	TRUE	1900000000	sessionid	ABC123...
.instagram.com	TRUE	/	TRUE	1900000000	csrftoken	XYZ789...
```

## コード

```bash
USERNAME='target_account'

# Cookie の csrftoken と一致した値を X-CSRFToken に入れる
CSRF=$(awk '$6=="csrftoken"{print $7}' cookies.txt | tail -1)

curl -b cookies.txt \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36' \
  -H 'sec-ch-ua: "Chromium";v="131", "Not_A Brand";v="24"' \
  -H 'sec-ch-ua-mobile: ?0' \
  -H 'sec-ch-ua-platform: "macOS"' \
  -H 'X-IG-App-ID: 936619743392459' \
  -H 'X-ASBD-ID: 129477' \
  -H 'X-CSRFToken: '"$CSRF" \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Accept: application/json, text/plain, */*' \
  -H 'Accept-Language: ja' \
  -H 'Referer: https://www.instagram.com/' \
  "https://www.instagram.com/api/v1/users/web_profile_info/?username=${USERNAME}" \
  -o profile.json -w 'http_code=%{http_code} size=%{size_download}\n'
```

## 期待出力

- 標準出力: `http_code=200 size=15234`
- `profile.json` は通常 5KB〜50KB の JSON。`{"data":{"user":{"username":"...","biography":"...","edge_owner_to_timeline_media":{"count":...,"edges":[...]}}}}` 形式
- `http_code=403 / 401`: Cookie 期限切れまたは header 不足
- `http_code=302`: ログイン画面へのリダイレクト

## ハマりポイント

- `X-IG-App-ID: 936619743392459` は Instagram Web の公開固定値。他 SNS は値が異なる
- 同 IP から短時間に大量リクエストすると一時 ban。1 req/秒以下に抑制
- `sessionid` の期限は約 90 日。期限切れの兆候: HTTP 302 → `/accounts/login/`
- `cookies.txt` を git に commit すると session 漏洩。`.gitignore` 必須
- 2024 年以降、Instagram は `web_profile_info` への匿名アクセスを大幅に制限している。Cookie + UA だけでは 401/403 になることが増えた。上のヘッダ（`X-CSRFToken` / `X-ASBD-ID` / `X-Requested-With` / `sec-ch-ua-*`）を揃えても通らない場合は、JA3/JA4 TLS fingerprint まで合わせる必要がある → [curl-impersonate](https://github.com/lwthiker/curl-impersonate)（`curl_chrome116` 等）に置き換える、または C の特化 OSS（instaloader）にフォールバック
- UA の Chrome バージョンは `sec-ch-ua` のバージョンと一致させる。古い Chrome 120 を装って `sec-ch-ua: "Chromium";v="131"` を返すと不整合で検知される
