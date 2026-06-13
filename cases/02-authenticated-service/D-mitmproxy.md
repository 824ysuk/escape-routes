# 事例 2-D: mitmproxy で観察 → 再現

> **対象範囲**: 対象サービスの観察に関与しない通信を傍受しない構成にする。system-wide proxy + 信頼した CA は、その PC で動作する全アプリ (Slack / 1Password / 業務 SaaS / VPN クライアント) の TLS を MITM 可能にする。**会社支給 PC では社内情報セキュリティポリシーを確認**。可能なら system-wide proxy ではなく `--mode reverse:https://app.example.com` または per-application proxy を使う。

## 前提 / install

```bash
brew install mitmproxy
# または
pip install mitmproxy
mitmproxy --version

chmod 700 ~/.mitmproxy    # CA 秘密鍵保護
```

- proxy 設定（OS 別）:
    - **macOS**: 環境設定 → ネットワーク → 詳細 → プロキシ → Web プロキシ HTTP / 安全な Web プロキシ HTTPS 両方を `127.0.0.1:8080`
    - **Windows**: 設定 → ネットワークとインターネット → プロキシ → 手動で `127.0.0.1:8080`
    - **Linux GNOME**: 設定 → ネットワーク → ネットワーク プロキシ → 手動
- CA 証明書 install:
    - **macOS**: `http://mitm.it` から `mitmproxy-ca-cert.pem` を DL → Keychain Access で「常に信頼」に変更
    - **Linux**: DL 後 `sudo cp mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/mitmproxy.crt && sudo update-ca-certificates`
    - **Windows**: DL した `.p12` を実行 → 「ローカル コンピューター」→「信頼されたルート証明機関」に install

## proxy mode の選び方

| mode | 範囲 | 用途 |
|---|---|---|
| `regular` (default) | system-wide proxy 設定の全通信 | 対象サービスのみ動かす状況 |
| `--mode reverse:https://app.example.com` | 特定ドメインのみ | 業務 PC で他通信を傍受したくないとき推奨 |
| `--mode transparent` | iptables 経由の透過 proxy | 共有 NAT 等 |
| `--mode upstream:http://corp-proxy:8080` | 既存 corp proxy の前段 | corporate network での chain |

業務 PC では `--mode reverse` を default にする。

## コード

```bash
# 1. mitmweb を起動（Web UI 付き、対象ドメインのみ）
mitmweb --listen-port 8080 --mode reverse:https://app.example.com
# → http://127.0.0.1:8081 が別タブで自動で開く

# system-wide proxy で観察したい場合
# mitmweb --listen-port 8080

# 2. ブラウザで対象サービスにログイン → 取得したい操作を実行
# 3. mitmweb の Web UI で対象 flow をクリック → Edit → "Copy as → curl"

# 4. クリップボードにコピーされた curl の例 (機密値は --config 経由が安全):
set +o history
SESSION='session=<value>'
CSRF_HDR='X-CSRF-Token: <value>'
curl --config - <<EOF >orders.json
url = "https://app.example.com/api/v1/orders"
header = "Cookie: ${SESSION}"
header = "${CSRF_HDR}"
header = "Accept: application/json"
write-out = "http_code=%{http_code}\\n"
EOF
```

## 期待出力

- mitmweb の Web UI（`http://127.0.0.1:8081`）に対象ブラウザの HTTP / HTTPS リクエストが flow として一覧表示
- 各 flow をクリック → Response タブで body 確認可能
- "Copy as → curl" の curl を実行すると `http_code=200` + 同じレスポンス

## ハマりポイント

### 観察したリクエストをそのまま curl 化できない値

mitmproxy で見える HTTP には **one-shot な値** が多数含まれる。そのまま curl 化すると 2 回目以降 fail する:

| 値 | 性質 | 再現方法 |
|---|---|---|
| `X-CSRF-Token` | session 単位の rotation を伴うことが多い | HTML から再取得が必要 |
| OAuth `state` / `nonce` | 1 リクエスト限り | CSRF 防御 (RFC 9700 §4.7) のため再生成必須 |
| PKCE `code_verifier` / `code_challenge` | ペアで成立 | 再生成しないなら認可コード再要求も無効 |
| `__Host-` / `__Secure-` prefix cookie | HTTPS + path / 必須 | HTTPS で送る |

**観察 → 再現は「ハートビート / 一覧 GET 等の冪等 read」に向き、認可フロー全体の再現には向かない**。

### proxy 設定の後始末

- 終了時に各 OS の proxy 設定を OFF に戻すこと。忘れると mitmproxy 停止後に通信不能
- 機械的に確認するコマンド (設定ファイル上 OFF でも有効化されている場合あり):
  ```bash
  scutil --proxy                                # macOS
  networksetup -getwebproxy Wi-Fi               # macOS
  gsettings get org.gnome.system.proxy mode     # Linux GNOME
  reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable  # Windows
  ```

### CA の削除手順 (使用後)

mitmproxy 終了後は **PC 側の CA も削除する**。残置すると CA 秘密鍵を取得した攻撃者が任意 HTTPS を MITM 可能:

- **macOS**: Keychain Access → mitmproxy 証明書を選択 → delete
- **Linux**: `sudo rm /usr/local/share/ca-certificates/mitmproxy.crt && sudo update-ca-certificates --fresh`
- **Windows**: `certmgr.msc` → 信頼されたルート証明機関 → 証明書 → mitmproxy → delete

CA 秘密鍵 `~/.mitmproxy/mitmproxy-ca.pem` を流出させない (`chmod 700 ~/.mitmproxy`)。

### TLS Certificate Pinning

- Pinning が効いているアプリ (Twitter / 銀行系など) では復号できない
- **自社アプリ / 認可された pentest 対象** であれば Frida + Objection で pinning bypass が定石 ([OWASP MASTG](https://mas.owasp.org/MASTG/) 参照)
- 他社アプリへの bypass は不正競争防止法 2 条 1 項 10 号 / 対象アプリ EULA / DMCA Section 1201 に触れうる

### その他

- `http://mitm.it` にアクセス不能: proxy 設定が向いていない or mitmproxy が起動していない
- "Copy as → curl" で得たコマンドには Cookie / CSRF token が平文で含まれる。script 化して保存するなら値は `--config -` heredoc 経由で渡す ([02-A-cookie-curl.md](A-cookie-curl.md) 参照)
- 詳細は [mitmproxy: Certificate concepts](https://docs.mitmproxy.org/stable/concepts-certificates/) / [mitmproxy: Modes](https://docs.mitmproxy.org/stable/concepts-modes/)
