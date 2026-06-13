# 事例 5-A: mitmproxy でモバイルアプリの通信を傍受

> **対象範囲**: 自分の端末 + 自分のアカウント + 自社が運用するアプリ、または書面で復号同意のある対象に限る。他社アプリの TLS pinning bypass / APK patch は対象アプリ EULA + 各国法に触れうる。共用 Wi-Fi では同一 LAN 上の他人の通信も技術的には傍受できる構成のため、自宅・個室 Wi-Fi で実行する。日本: 電気通信事業法 4 条 (通信の秘密)、米国: Wiretap Act 18 U.S.C. § 2511、UK: Investigatory Powers Act 2016 §3。

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

### 3 つの frontend の使い分け

| frontend | 用途 |
|---|---|
| `mitmweb` | Web UI で対話的に観察 (本ファイルの例) |
| `mitmproxy` | TUI で対話的に観察 (headless / SSH 経由) |
| `mitmdump` | CLI / scriptable、CI 自動化用 |

## コード

```bash
# 1. mitmweb を起動（Web UI 付き）
mitmweb --listen-port 8080
# → http://127.0.0.1:8081 が別タブで自動で開く

# 2. モバイル端末側で Wi-Fi 設定 → 詳細設定 → HTTP プロキシ → 手動
#    サーバー = <PC の IP>、ポート = 8080

# 3. モバイルブラウザで http://mitm.it を開いて CA を取得
#    iOS (2 段階手順、両方必要):
#      Step 1: 設定 → 一般 → VPN とデバイス管理 → mitmproxy → インストール
#      Step 2: 設定 → 一般 → 情報 → 証明書信頼設定 → mitmproxy CA を ON
#      ※ Step 2 を忘れると証明書はインストール済みに見えるが TLS handshake で
#         bad certificate になり HTTPS が NSURLErrorDomain -1202 で失敗する
#    Android: 設定 → セキュリティ → 暗号化と認証情報 → 証明書をインストール → CA 証明書

# 4. 対象アプリを起動 → mitmweb の Web UI でリクエスト一覧
# 5. 該当 flow を右クリック → Replay / Modify / "Export as → curl"
```

## 期待出力

- mitmweb の Web UI（`http://127.0.0.1:8081`）に対象アプリの HTTP / HTTPS リクエストが flow として一覧表示（method / URL / status / body）
- CA 信頼後は HTTPS body も復号して中身が見える
- Export as curl したコマンドが PC で実行できる

## ハマりポイント

### 復号できないケース

- **TLS Certificate Pinning** が効いているアプリ（Twitter / 銀行系など）では復号できない。**自社アプリ / 認可された pentest 対象** であれば [Frida](https://frida.re/) + [Objection](https://github.com/sensepost/objection) で pinning bypass が定石 ([OWASP MASTG: Testing Network Communication](https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0036/) 参照)。他社アプリへの bypass は対象アプリ EULA / DMCA Section 1201 / 不正競争防止法 2 条 1 項 10 号 (技術的制限手段の回避) に触れうる
- **HTTP/3 (QUIC over UDP/443) は OS の HTTP proxy 設定では intercept されない**。Twitter / Google / Meta / Cloudflare で広く有効。回避は (a) Chrome なら `chrome://flags#enable-quic` を OFF、(b) アプリ側で Alt-Svc を無視する設定、(c) mitmproxy 11+ の `--mode reverse:quic://...` を別途構成 ([RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000))

### Android の CA 信頼条件

- **`targetSdkVersion >= 24` (Android 7.0+) の APK は端末追加 CA を信頼しない**。古い APK (`targetSdkVersion <= 23`) は Android 14 上でも user-installed CA を信頼する
- 自社アプリは [`network_security_config.xml`](https://developer.android.com/training/articles/security-config) で許可:
  ```xml
  <!-- debug build 限定なら base-config ではなく debug-overrides に -->
  <network-security-config>
    <debug-overrides>
      <trust-anchors>
        <certificates src="user" />
      </trust-anchors>
    </debug-overrides>
  </network-security-config>
  ```

### proxy 設定の後始末

- mitmproxy 終了後はモバイル端末の Wi-Fi proxy 設定を「OFF」または「自動」に戻さないと以後の通信が全部死ぬ
- macOS / Linux 側で proxy を立てた場合は機械的に確認:
  ```bash
  scutil --proxy                                # macOS
  networksetup -getwebproxy Wi-Fi               # macOS
  gsettings get org.gnome.system.proxy mode     # Linux GNOME
  ```

### mitmproxy CA の取り扱い

- **使用後は端末から CA を必ず削除する**。残置すると、その CA 秘密鍵を取得した攻撃者が該当端末で任意の HTTPS を MITM 可能
  - iOS: 設定 → 一般 → VPN とデバイス管理 → mitmproxy → プロファイル削除
  - Android: 設定 → セキュリティ → 暗号化と認証情報 → ユーザー認証情報 → 削除
- PC 側の CA 秘密鍵 `~/.mitmproxy/mitmproxy-ca.pem` を流出させない (`chmod 700 ~/.mitmproxy`)
- Certificate Transparency ([RFC 9162](https://datatracker.ietf.org/doc/html/rfc9162)) は private CA を要求対象外にするため、端末注入された CA は CT log に出ない。これが副作用の温床

### TLS 1.3 復号の仕組み

mitmproxy は TLS 1.3 ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)) を full handshake で書き換えて自前 CA で再署名する MITM プロキシ。TLS resumption / 0-RTT は対応制約があり、resumption 多用サーバーでパフォーマンス・互換性問題が出る場合がある。
