# 事例 2-B: 公式 API + OAuth Device Flow

> **OAuth 規範の更新**: OAuth 2.0 は [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) (2012) を起点とするが、現在の推奨は [RFC 9700 OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/rfc9700/) (2024-12) と [OAuth 2.1 draft](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)。本手順も RFC 9700 §2 (PKCE 必須化、implicit/ROPC 廃止) に整合させている。

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
# 0. 機密値が shell history に残らないように
set +o history
umask 077                          # 新規ファイルを 600 で作る

# 1. デバイスコードを取得
curl -sS -X POST 'https://oauth.example.com/device/code' \
  -d "client_id=$CLIENT_ID" \
  -d 'scope=read:orders' \
  -o device.json
cat device.json
# 応答例: {"device_code":"GmRhmh...","user_code":"WDJB-MJHT",
#         "verification_uri":"https://example.com/device",
#         "verification_uri_complete":"https://example.com/device?code=WDJB-MJHT",
#         "interval":5,"expires_in":900}

# 2. ユーザーに表示する URL を verification_uri_complete 優先で組み立てる
#    (RFC 8628 §3.3.1: complete URI は QR コード等で user_code 入力を省ける)
jq -r '"Open: \(.verification_uri_complete // .verification_uri)\nCode:  \(.user_code)"' device.json

# 3. polling (RFC 8628 §3.5)
DEVICE_CODE=$(jq -r .device_code device.json)
INTERVAL=$(jq -r '.interval // 5' device.json)   # RFC 8628 §3.2: 省略時 5 秒
EXPIRES_IN=$(jq -r .expires_in device.json)
DEADLINE=$((SECONDS + EXPIRES_IN))

while (( SECONDS < DEADLINE )); do
  curl -sS -X POST 'https://oauth.example.com/token' \
    -d 'grant_type=urn:ietf:params:oauth:grant-type:device_code' \
    -d "device_code=$DEVICE_CODE" \
    -d "client_id=$CLIENT_ID" \
    -o token.json
  if jq -e .access_token token.json > /dev/null 2>&1; then break; fi
  ERR=$(jq -r .error token.json)
  case "$ERR" in
    authorization_pending) ;;
    slow_down)
      # RFC 8628 §3.5: polling 間隔を増加させる義務。上限は 60 秒
      INTERVAL=$(( INTERVAL < 60 ? INTERVAL + 5 : INTERVAL ))
      ;;
    expired_token|access_denied)
      echo "終了: $ERR" >&2
      exit 1
      ;;
    *)
      echo "未知のエラー: $ERR" >&2
      exit 2
      ;;
  esac
  sleep "$INTERVAL"
done

if ! jq -e .access_token token.json > /dev/null 2>&1; then
  echo "expires_in (${EXPIRES_IN}s) 経過。ユーザーが verification_uri を完了していない可能性" >&2
  exit 3
fi

# 4. access_token で protected API を呼ぶ
#    (RFC 6750 §2.1: Bearer は必ず Authorization header で送る。query / body は log / referrer leak の原因)
ACCESS_TOKEN=$(jq -r .access_token token.json)
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  'https://api.example.com/orders' \
  -o orders.json -w 'http_code=%{http_code}\n'

# 5. refresh で延命 (RFC 9700 §2.2.2: refresh_token rotation 対応)
REFRESH_TOKEN=$(jq -r .refresh_token token.json)
curl -sS -X POST 'https://oauth.example.com/token' \
  -d 'grant_type=refresh_token' \
  -d "refresh_token=$REFRESH_TOKEN" \
  -d "client_id=$CLIENT_ID" \
  -o token-new.json

if jq -e .access_token token-new.json > /dev/null 2>&1; then
  # rotation 対応: 新しい refresh_token があれば書き戻す
  NEW_REFRESH=$(jq -r '.refresh_token // empty' token-new.json)
  if [ -n "$NEW_REFRESH" ]; then
    mv token-new.json token.json
  fi
elif [ "$(jq -r .error token-new.json)" = "invalid_grant" ]; then
  # rotation 検知の可能性。漏洩 trigger として全 token を revoke する
  echo "invalid_grant: rotation 検知の可能性。全 token を revoke する" >&2
  curl -sS -X POST 'https://oauth.example.com/revoke' \
    -d "token=$REFRESH_TOKEN" \
    -d "client_id=$CLIENT_ID"
  exit 4
fi
```

## 期待出力

- `device.json`: `{"device_code":"GmRhmh...","user_code":"WDJB-MJHT","verification_uri":"...","verification_uri_complete":"...","interval":5,"expires_in":900}`
- 認証完了後 `token.json`: `{"access_token":"eyJ...","refresh_token":"...","expires_in":3600,"token_type":"Bearer","scope":"read:orders"}`
- `orders.json` に protected API の response（数 KB〜数百 KB）

## ハマりポイント

### polling と back-off

- polling 間隔は `interval` 秒以上 (RFC 8628 §3.5)。`slow_down` 受信時は increment 必須だが、増分は実装依存。Google / GitHub は +5s が慣例。上限を設けないと `expires_in` 内に rate limit と競合する
- `interval` が省略された場合の fallback は 5 秒 (RFC 8628 §3.2)

### Bearer / refresh_token の取り扱い

- `access_token` の `expires_in` は 1 時間程度が典型。`refresh_token` で再取得する処理が運用必須
- **refresh_token rotation** (RFC 9700 §2.2.2 / RFC 6749 §10.4): public client では rotation + reuse detection が事実上必須。response に新しい `refresh_token` があれば古いものを上書きする
- `invalid_grant` が返ったら漏洩検知 trigger の可能性。即座に [RFC 7009 token revocation](https://datatracker.ietf.org/doc/html/rfc7009) (`/revoke`) を叩いて全 token を無効化する
- **Bearer は `Authorization: Bearer` header でのみ送る** (RFC 6750 §2.1)。URL query / form body 経由は log / referrer / proxy 経由で漏れる。RFC 9700 でも非推奨

### PKCE

- Device Flow 自体は PKCE 不要 (RFC 8628 仕様)
- ただし RFC 9700 / OAuth 2.1 では **全 client 種別で PKCE は MUST**。authorization code flow を別途使う場合は code_verifier / code_challenge を必ず付ける
- Device Flow でも verification_uri に対する man-in-the-browser 攻撃が成立しうるため、可能なら別チャネル (signed QR / push) で user_code を確認する

### Token 漏洩経路

- `token.json` / `device.json` には access_token / refresh_token が平文で残る
  - **`.gitignore` に追加** (トップ `.gitignore` 共通エントリ参照)
  - `chmod 600 token.json` (上記 `umask 077` で自動)
  - 長期保管は OS の secret store: macOS `security add-generic-password`、Linux `secret-tool store`、CI は OIDC federated credential
- `curl -d "client_secret=..."` の引数値は `ps aux` で見えるため、機密値は `--config -` の heredoc 経由で渡す:
  ```bash
  curl --config - <<EOF
  url = "https://oauth.example.com/token"
  data = "client_secret=$CLIENT_SECRET"
  EOF
  ```
- `set -x` / `bash -x` 系のデバッグで token が log に流れる。sensitive block は `set +x` で囲う

### 発展

次に学ぶべき OAuth トピック:

- [RFC 9126 Pushed Authorization Requests (PAR)](https://datatracker.ietf.org/doc/html/rfc9126)
- [RFC 9449 OAuth 2.0 Demonstrating Proof of Possession (DPoP)](https://datatracker.ietf.org/doc/html/rfc9449) — sender-constrained Bearer
- [RFC 9700 §2.2 Sender-constrained Bearer の必要性](https://datatracker.ietf.org/doc/html/rfc9700#section-2.2)
