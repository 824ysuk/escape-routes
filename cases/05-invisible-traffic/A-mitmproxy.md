# 事例 5-A: mitmproxy でモバイルアプリの通信を傍受

## 前提 / install

```bash
brew install mitmproxy   # macOS
pip install mitmproxy    # クロスプラットフォーム

# PC の IP を確認
ipconfig getifaddr en0   # macOS の Wi-Fi
ip addr show             # Linux
ipconfig                 # Windows
```

- モバイル端末を PC と同じ Wi-Fi に接続
- **モバイルデータ通信（Cellular）は proxy を経由しない**: Wi-Fi のみ proxy 設定する仕様のため、4G/5G に切り替わるとアプリが直接通信して観察不能。機内モード + Wi-Fi のみ ON で確実化

## コード

```bash
# 1. mitmweb を起動（Web UI 付き）
mitmweb --listen-port 8080
# → http://127.0.0.1:8081 が別タブで自動で開く

# 2. モバイル端末側で Wi-Fi 設定 → 詳細設定 → HTTP プロキシ → 手動
#    サーバー = <PC の IP>、ポート = 8080

# 3. モバイルブラウザで http://mitm.it を開く
#    iOS: 設定 → 一般 → VPN とデバイス管理 → mitmproxy → インストール
#         さらに 設定 → 一般 → 情報 → 証明書信頼設定 で mitmproxy CA を ON
#    Android: 設定 → セキュリティ → 暗号化と認証情報 → 証明書をインストール → CA 証明書

# 4. 対象アプリを起動 → mitmweb の Web UI でリクエスト一覧
# 5. 該当 flow を右クリック → Replay / Modify / "Export as → curl"
```

## 期待出力

- mitmweb の Web UI（`http://127.0.0.1:8081`）に対象アプリの HTTP / HTTPS リクエストが flow として一覧表示（method / URL / status / body）
- CA 信頼後は HTTPS body も復号して中身が見える
- Export as curl したコマンドが PC で実行できる

## ハマりポイント

- **TLS Certificate Pinning** が効いているアプリ（Twitter / 銀行系など）では復号できない。Frida + Objection で pin removal が必要（侵襲度高）
- **Android 7+ の APK は端末追加 CA を信頼しない仕様**。自社アプリなら build 時に [`network_security_config.xml`](https://developer.android.com/training/articles/security-config) で許可。他社アプリは APK patch が必要
- **Wi-Fi 専用**: モバイルデータに切り替わると proxy 経由しない → 観察中は端末を機内モード + Wi-Fi ON で運用
- **mitmproxy 終了後の後片付け**: モバイル端末の Wi-Fi proxy 設定を「OFF」または「自動」に戻さないと以後の通信が全部死ぬ
