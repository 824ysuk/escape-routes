# 事例 1-B: Cookie + curl + UA 偽装

> **対象範囲**: 対象アカウントが利用者自身のもの、または公開アカウントの公開情報に限る。第三者の非公開コンテンツや大規模収集は Meta / X / TikTok の ToS 違反 + 不正競争防止法 / hiQ Labs v. LinkedIn 判決後も認証 bypass 系は CFAA 適用範囲。Meta v. BrandTotal (N.D. Cal. 2022) で認証済み session を使った scraping が違法と認定された先例あり。

## 前提 / install

- ブラウザで対象サイトにログイン
- Cookie 抽出は拡張機能を使う:
    - Chrome / Edge: [Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc) (Netscape 形式 export、ローカル処理のみ)
    - Firefox: [cookies.txt](https://addons.mozilla.org/firefox/addon/cookies-txt/)
    - **EditThisCookie は 2024 年に開発者交代後 malicious adware の混入が報告され Web Store から取り下げられた経緯がある (旧 ID `fngmhnnpilhplaeedifhccceomclgfbg`)**。後継 V3 も signal が混乱しているため `Get cookies.txt LOCALLY` を推奨
- 手動の場合: DevTools → Application → Cookies → 全 cookie をテキストに転写
- curl install: macOS / Linux はプリインストール、Windows は `winget install curl.curl`
- `cookies.txt` は **必ず `.gitignore` に追加**（session 情報のため共有禁止、共通 patterns はトップ `.gitignore` 参照）
- ファイル権限を 600 に: `chmod 600 cookies.txt`

Netscape 形式の `cookies.txt`（TAB 区切り 7 フィールド = domain / include subdomain (TRUE/FALSE) / path / secure (TRUE/FALSE) / expiration (UNIX timestamp) / name / value）:

```
# Netscape HTTP Cookie File
.instagram.com	TRUE	/	TRUE	1900000000	sessionid	ABC123...
.instagram.com	TRUE	/	TRUE	1900000000	csrftoken	XYZ789...
.instagram.com	TRUE	/	TRUE	1900000000	ds_user_id	1234567890
.instagram.com	TRUE	/	TRUE	1900000000	mid	XXXXXXXXXXXX
```

Instagram の最小必須 cookie: `sessionid`, `csrftoken`, `ds_user_id`, `mid` (Machine ID, 数年寿命)。`mid` を捨てると別 device 扱いで shadowban の発火率が上がる。

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
- `http_code=403 / 401`: Cookie 期限切れまたは header 不足、または JA3/H2 fingerprint で検知 (下記)
- `http_code=302`: ログイン画面へのリダイレクト

## ハマりポイント

### 素 curl では 2024 以降 Instagram で 403 が頻発する (TLS / HTTP/2 fingerprint)

Cloudflare / Akamai / DataDome は TLS ClientHello から [JA3](https://github.com/salesforce/ja3) (Salesforce 2017) / [JA4](https://github.com/FoxIO-LLC/ja4) (FoxIO 2023) を抽出する。素 curl は OpenSSL/LibreSSL 由来の TLS 指紋が curl 固有で、UA を 100 通り変えても TLS 指紋は一意。Akamai は加えて HTTP/2 の SETTINGS / WINDOW_UPDATE / HEADERS の順序・priority 値を組み合わせた h2 fingerprint も評価する。

**回避**: [curl-impersonate](https://github.com/lwthiker/curl-impersonate) (C 本体) または [curl_cffi](https://github.com/yifeikong/curl_cffi) (Python wrapper) で BoringSSL 経由の Chrome impersonation:

```python
# pip install curl_cffi
from curl_cffi import requests

cookies = {'sessionid': '...', 'csrftoken': '...', 'ds_user_id': '...', 'mid': '...'}
r = requests.get(
    f'https://www.instagram.com/api/v1/users/web_profile_info/?username={USERNAME}',
    cookies=cookies,
    impersonate='chrome124',  # JA3/JA4/H2 fingerprint が Chrome 124 と一致
    headers={'X-IG-App-ID': '936619743392459'},
)
```

これだけで TLS / H2 指紋が Chrome と一致する。

### 必須ヘッダの拡張 (private endpoint で 401 になる場合)

`/api/v1/users/web_profile_info/` は緩い endpoint だが、他 endpoint (`/api/v1/feed/...` 等) では追加ヘッダが要る:

| Header | 値の例 | 備考 |
|---|---|---|
| `X-IG-App-ID` | `936619743392459` | Web 固定 |
| `X-ASBD-ID` | `198387` | 時期で変動。DevTools Network で copy |
| `X-IG-WWW-Claim` | `0` 初回 → response header `x-ig-set-www-claim` から rotate |
| `X-Web-Session-ID` | `<16進16桁>` | session 単位 |
| `X-Requested-With` | `XMLHttpRequest` |
| `X-CSRFToken` | cookie の `csrftoken` と同値 |

実践的指針: **DevTools Network で実際のリクエストを Copy as cURL したものを base に変形する**。

### Cookie 漏洩防止

- 同 IP から短時間に大量リクエストすると一時 ban。1 req/秒以下に抑制
- `sessionid` の期限は約 90 日。期限切れの兆候: HTTP 302 → `/accounts/login/`
- `cookies.txt` を git に commit すると session 漏洩。`.gitignore` 必須 (CONTRIBUTING.md の Secret hygiene 参照)
- 万一 commit した場合は **対象サービスの「他端末からログアウト」を先に実行** → git history 削除の順 (逆順では rotate 前に攻撃者が利用する)

### 他 SNS の値

`X-IG-App-ID: 936619743392459` は Instagram Web の公開固定値。他 SNS は値が異なる。X (Twitter): `Authorization: Bearer AAAA...` + `x-csrf-token` + `x-twitter-auth-type: OAuth2Session` ほか。

### 関連事例

- 検知の全体像と多軸選択は事例 1 README の「検知の層」「他手段を選ぶ条件」を参照
- 公式 API で済むなら → [D-official-api.md](D-official-api.md) (Instagram Graph API、自分の Business アカウント限定)
