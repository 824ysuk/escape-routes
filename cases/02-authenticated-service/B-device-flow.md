# 事例 2-B: 公式 API + OAuth Device Flow

## 前提 / install

- 対象サービスが [Device Authorization Grant (RFC 8628)](https://datatracker.ietf.org/doc/html/rfc8628) をサポートしていることを確認
- 代表的プロバイダー:
    - Google: [OAuth 2.0 設定](https://support.google.com/googleapi/answer/6158849)
    - Auth0: ダッシュボード → Applications → "Native" タイプで作成 → Grant Types で "Device Code" を有効化
    - Okta: API → Authorization Servers → ポリシーで Device Authorization を許可
    - GitHub: Settings → Developer settings → OAuth Apps → "Enable Device Flow" にチェック
- それぞれ `client_id` を取得 → 環境変数 `CLIENT_ID` に保存
- `jq` install: `brew install jq` / `apt install jq`
- スコープ命名規則はプロバイダー依存（例: `read:orders` / `https://www.googleapis.com/auth/drive.readonly`）

## コード

```bash
# 1. デバイスコードを取得
curl -X POST 'https://oauth.example.com/device/code' -d "client_id=$CLIENT_ID" -d 'scope=read:orders' -o device.json
cat device.json
# 応答例: {"device_code":"GmRhmh...","user_code":"WDJB-MJHT","verification_uri":"https://example.com/device","interval":5,"expires_in":900}

# 2. ユーザーがブラウザで verification_uri に user_code を入力（CLI は polling）
DEVICE_CODE=$(jq -r .device_code device.json)
INTERVAL=$(jq -r .interval device.json)
while true; do
  curl -X POST 'https://oauth.example.com/token' -d 'grant_type=urn:ietf:params:oauth:grant-type:device_code' -d "device_code=$DEVICE_CODE" -d "client_id=$CLIENT_ID" -o token.json
  if jq -e .access_token token.json > /dev/null 2>&1; then break; fi
  ERR=$(jq -r .error token.json)
  case "$ERR" in
    authorization_pending) ;;
    slow_down)             INTERVAL=$((INTERVAL + 5)) ;;
    expired_token|access_denied) echo "終了: $ERR"; exit 1 ;;
  esac
  sleep "$INTERVAL"
done

# 3. access_token で protected API を呼ぶ
ACCESS_TOKEN=$(jq -r .access_token token.json)
curl -H "Authorization: Bearer $ACCESS_TOKEN" 'https://api.example.com/orders' -o orders.json -w 'http_code=%{http_code}\n'

# 4. refresh で延命
REFRESH_TOKEN=$(jq -r .refresh_token token.json)
curl -X POST 'https://oauth.example.com/token' -d 'grant_type=refresh_token' -d "refresh_token=$REFRESH_TOKEN" -d "client_id=$CLIENT_ID" -o token.json
```

## 期待出力

- `device.json`: `{"device_code":"GmRhmh...","user_code":"WDJB-MJHT","verification_uri":"https://...","interval":5,"expires_in":900}`
- 認証完了後 `token.json`: `{"access_token":"eyJ...","refresh_token":"...","expires_in":3600,"token_type":"Bearer","scope":"read:orders"}`
- `orders.json` に protected API の response（数 KB〜数百 KB）

## ハマりポイント

- polling 間隔は `interval` 秒以上。`slow_down` 受信時は `interval` を 5 秒ずつ増やす（上のコードに実装済み）
- `access_token` の `expires_in` は 1 時間程度が典型。`refresh_token` で再取得する処理が運用必須
- **PKCE は Device Flow では任意**（RFC 8628 仕様）。authorization code flow では推奨
- `token.json` には access_token / refresh_token が平文で残る。`cookies.txt` 同様 **`.gitignore` に追加** し、漏洩時は即 revoke
