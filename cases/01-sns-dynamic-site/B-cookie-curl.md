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

curl -b cookies.txt \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36' \
  -H 'X-IG-App-ID: 936619743392459' \
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
