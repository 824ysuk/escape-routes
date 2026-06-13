# 事例 2-A: Cookie 抽出 + curl

> **対象範囲**: 対象は利用者自身のアカウント、または自社が運用するサービスに限る。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 前提 / install

- ブラウザで対象サービスにログイン (2FA 含む完全ログイン)
- Cookie 抽出と Netscape 形式の cookies.txt 生成手順は **[事例 1-B](../01-sns-dynamic-site/B-cookie-curl.md) と同じ**（[Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc) 拡張機能、または DevTools → Application → Cookies の手動転写）
- `cookies.txt` は **必ず `.gitignore` に追加** (トップ `.gitignore` 共通エントリ参照)
- ファイル権限を 600 に: `chmod 600 cookies.txt`

## Cookie 属性の前提

ブラウザから抽出する前に、対象 cookie の属性を確認する。

| 属性 | 意味 | curl での扱い |
|---|---|---|
| **HttpOnly** | JS から読めない | ブラウザ拡張 / DevTools 経由でのみ取得可能 (JS console では取れない) |
| **Secure** | HTTPS のみ送信 | 対象 URL を `https://` で指定する必要 |
| **SameSite=Strict** | cross-site では送信されない | curl では関係なし。ブラウザ自動化 (Playwright) で omit される罠 |
| **SameSite=Lax** | cross-site の top-navigation GET でのみ送信 | 同上 |
| `__Host-` prefix | `Secure` + `Path=/` + Domain なし必須 | https + path / で送る |
| `__Secure-` prefix | `Secure` 必須 | https で送る |

詳細: [OWASP Session Management Cheat Sheet §Cookies](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#cookies)

## コード

```bash
# 機密値が history に残らないように
set +o history
umask 077                    # 新規ファイルを 600 で作る
chmod 600 cookies.txt

CSRF_TOKEN='<value>'         # ここでは見せるが、本来は次の --config 経由で渡すべき
UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36'

# 安全な渡し方: --config 経由 (機密値が argv / ps に出ない)
curl -b cookies.txt --config - <<EOF >orders.json
url = "https://internal.example.com/api/v1/orders"
user-agent = "${UA}"
header = "Accept: application/json"
header = "X-CSRF-Token: ${CSRF_TOKEN}"
header = "X-Requested-With: XMLHttpRequest"
write-out = "http_code=%{http_code} size=%{size_download}\\n"
EOF

# どうしても -H で書く必要があるなら、機密値だけ env 経由 + curl の env 展開
# curl -b cookies.txt -H "User-Agent: ${UA}" -H "X-CSRF-Token: ${CSRF_TOKEN}" ...
# ※ -H の値は ps -ef で argv に出るため CI / 共有 host では危険
```

## 期待出力

- 標準出力: `http_code=200 size=42153`
- `orders.json` 構造例: `{"orders":[{"id":"ord_001","total":1500,"created_at":"2026-06-11T10:00:00Z"}],"total_count":123,"has_next":true}`
- `http_code=401 / 403`: Cookie 期限切れ → 再ログインして取り直し
- `http_code=302`: ログイン画面 redirect。`curl -L -v` で追跡

## ハマりポイント

### Bearer / CSRF / Cookie の安全な渡し方

- **`-H "Authorization: ..."` は argv に乗り `ps -ef` で見える**。`curl --config -` の heredoc 経由で渡す ([curl docs: passwords and credentials](https://everything.curl.dev/usingcurl/passwords.html))
- `set -x` / `bash -x` 系のデバッグで token が log に流れる。sensitive block は `set +x` / `set +o history` で囲う
- shell history に CSRF 値が残る → `HISTCONTROL=ignorespace` + 機密 line は leading space を付ける

### 漏洩時の rotation 手順

万一 `cookies.txt` を commit / 共有してしまった場合、**順序が重要**:

1. **対象サービスの「他端末からログアウト」を最初に実行**して該当 session を無効化 (例: [Google Security](https://myaccount.google.com/security), [GitHub Sessions](https://github.com/settings/sessions))
2. 同 session でアクセスされた可能性のあるリソース (リポジトリ・支払・メール) の操作ログを確認
3. 必要なら所属組織の SOC / IT 部門に通知
4. git history からの削除は session revoke の**後**に行う (順序を逆にすると rotate 前に攻撃者が利用する)

### Session lifetime

- session の有効期限はサービスにより数時間〜数週間。長期運用するなら [B (Device Flow)](B-device-flow.md) か [C (Playwright)](C-playwright.md) が安定
- CSRF token が必須のエンドポイント: DevTools → Network → 対象 request → Headers タブで「X-」プレフィックス header と Form Data を全部確認

### 参考

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [NIST SP 800-63B §7 Session Management](https://pages.nist.gov/800-63-3/sp800-63b.html#sec7)
