# 事例 2-D: mitmproxy で観察 → 再現

## 前提 / install

```bash
brew install mitmproxy
# または
pip install mitmproxy
mitmproxy --version
```

- proxy 設定（OS 別）:
    - **macOS**: 環境設定 → ネットワーク → 詳細 → プロキシ → Web プロキシ HTTP / 安全な Web プロキシ HTTPS 両方を `127.0.0.1:8080`
    - **Windows**: 設定 → ネットワークとインターネット → プロキシ → 手動で `127.0.0.1:8080`
    - **Linux GNOME**: 設定 → ネットワーク → ネットワーク プロキシ → 手動
- CA 証明書 install:
    - **macOS**: `http://mitm.it` から `mitmproxy-ca-cert.pem` を DL → Keychain Access で「常に信頼」に変更
    - **Linux**: DL 後 `sudo cp mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/mitmproxy.crt && sudo update-ca-certificates`
    - **Windows**: DL した `.p12` を実行 → 「ローカル コンピューター」→「信頼されたルート証明機関」に install
- 必ず使用後に proxy 設定を **OFF に戻す**

## コード

```bash
# 1. mitmweb を起動（Web UI 付き）
mitmweb --listen-port 8080
# → http://127.0.0.1:8081 が別タブで自動で開く

# 2. ブラウザで対象サービスにログイン → 取得したい操作を実行
# 3. mitmweb の Web UI で対象 flow をクリック → Edit → "Copy as → curl"

# 4. クリップボードにコピーされた curl の例:
SESSION='session=<value>'
CSRF_HDR='X-CSRF-Token: <value>'
curl 'https://app.example.com/api/v1/orders' -H "Cookie: ${SESSION}" -H "${CSRF_HDR}" -H 'Accept: application/json' -o orders.json -w 'http_code=%{http_code}\n'
```

## 期待出力

- mitmweb の Web UI（`http://127.0.0.1:8081`）に対象ブラウザの全 HTTP / HTTPS リクエストが flow として一覧表示
- 各 flow をクリック → Response タブで body 確認可能
- "Copy as → curl" の curl を実行すると `http_code=200` + 同じレスポンス

## ハマりポイント

- **TLS Certificate Pinning** が効いているアプリ（Twitter / 銀行系など）では復号できない
- **後始末**: 終了時に各 OS の proxy 設定を OFF に戻すこと。忘れると mitmproxy 停止後に通信不能
- `http://mitm.it` にアクセス不能: proxy 設定が向いていない or mitmproxy が起動していない
- "Copy as → curl" で得たコマンドには Cookie / CSRF token が平文で含まれる。script 化して保存するなら値は環境変数に逃がし、ファイルを git に commit しない
