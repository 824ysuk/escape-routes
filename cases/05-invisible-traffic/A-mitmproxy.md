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
#
# 注: Android 7 (API 24) 以降は targetSdk 24+ のアプリは user trust store を
#     無視するのが既定。ブラウザは復号できるがアプリ通信は復号できない。
#     アプリ通信を復号するには以下のいずれかが必要:
#       (a) 自社 APK: network_security_config.xml に
#           <trust-anchors><certificates src="user"/></trust-anchors>
#           を debug build に追加 ([Android Network Security Config](
#           https://developer.android.com/privacy-and-security/security-config))
#       (b) Emulator: emulator -writable-system で system trust store に push
#           ([mitmproxy: Install on Android](
#           https://docs.mitmproxy.org/stable/howto-install-system-trusted-ca-android/))
#       (c) 実機・他社 APK: apk-mitm で再パッケージ + Magisk +
#           MagiskTrustUserCerts 等

# 4. 対象アプリを起動 → mitmweb の Web UI でリクエスト一覧
# 5. 該当 flow を右クリック → Replay / Modify / "Export as → curl"
```

## 期待出力

- mitmweb の Web UI（`http://127.0.0.1:8081`）に対象アプリの HTTP / HTTPS リクエストが flow として一覧表示（method / URL / status / body）
- CA 信頼後は HTTPS body も復号して中身が見える
- Export as curl したコマンドが PC で実行できる

## ハマりポイント

- **TLS Certificate Pinning** が効いているアプリ（Twitter / 銀行系など）では復号できない。Frida + Objection で pin removal が必要（侵襲度高）
- **Android 7+ の APK は端末追加 CA を信頼しない仕様**。コードコメント参照
- **Wi-Fi 専用**: モバイルデータに切り替わると proxy 経由しない → 観察中は端末を機内モード + Wi-Fi ON で運用
- **mitmproxy 終了後の後片付けは 2 点セット**:
    1. モバイル端末の Wi-Fi proxy 設定を「OFF」または「自動」に戻す（しないと以後の通信が全部死ぬ）
    2. **CA 証明書の信頼解除**（残置するとその端末のあらゆる HTTPS 通信を後で MITM 可能な状態が残る）:
        - iOS: 設定 → 一般 → VPN とデバイス管理 → mitmproxy → プロファイル削除 + 設定 → 一般 → 情報 → 証明書信頼設定 で OFF
        - Android: 設定 → セキュリティ → 暗号化と認証情報 → ユーザー認証情報 → mitmproxy を削除
        - macOS: Keychain Access で mitmproxy 証明書を削除（または「常に拒否」へ）
- **export した curl / flow の取り扱い**: "Copy as → curl" で得たコマンドや `--set save_stream_file=...` で永続化した dump には Authorization / Cookie / Set-Cookie / PII が **全て平文** で残る。Slack / git / Issue に貼る前に Authorization と Cookie を環境変数化、レスポンス body の PII（氏名 / メールアドレス / 注文 ID 等）を `sed` 等でマスクする（[mitmproxy: overview features](https://docs.mitmproxy.org/stable/overview-features/#anticache)）
