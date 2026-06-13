# 事例 5: ブラウザに見えない通信を観察したい

> **対象範囲**: 自分の端末・自分のアカウント・自社が運用するアプリに限る。第三者の通信内容を傍受する行為は日本 電気通信事業法 4 条 (通信の秘密) / 米国 Wiretap Act / EU GDPR Art. 5・6 等に触れる。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 何が起きるか

ブラウザの DevTools Network タブで観察できるのは、そのブラウザが発した通信だけ。以下は見えない: モバイルアプリの通信、デスクトップアプリの通信、別端末からの通信、ブラウザ拡張内部の通信、暗号化されていて中身が読めない通信。

## 代替手段

| 手段 | 概要 | 用途 |
|---|---|---|
| [A. mitmproxy](A-mitmproxy.md) | ローカル proxy として動作、HTTPS を自前 CA で復号 | HTTPS API の観察、編集、再送 |
| [B. Charles Proxy / Proxyman](B-charles-proxyman.md) | GUI 版の HTTPS proxy（有料） | GUI で目視確認したい |
| [C. Wireshark / tshark](C-wireshark.md) | 生のパケットを観察。HTTPS は復号できないが SNI・TCP・TLS handshake は見える | プロトコル層のトラブル、低レベル観察 |
| [D. iOS Web Inspector / Android Studio Network Profiler](D-mobile-devtools.md) | 公式の OS / IDE ツール。CA インストール不要で安全 | 自社モバイルアプリのデバッグ |
| [E. アプリにデバッグログを仕込む](E-debug-log.md) | リクエスト直前にログを print | 1 リクエストだけ確認したい |

## 他手段を選ぶ条件

- **B（Charles / Proxyman）**: CLI が苦手、GUI で目視したい、有料ライセンスが許容できる
- **C（Wireshark）**: HTTPS の中身ではなく、コネクション失敗・TLS handshake error を見たい
- **D（公式 dev tool）**: 自社アプリで、CA インストールしたくない
- **E（デバッグログ）**: 1 リクエストだけ見たい、ソースが手元にある

## 補足

DevTools で見える「ブラウザの通信」は氷山の一角。プロトコル層を 1 段下げる（Wireshark）か、proxy を挟む（mitmproxy）か、アプリの中に入る（E）かで観察可能な領域が広がる。

### 2024-2026 のプロトコル進化への注意

- **TLS 1.3 ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446))** が広く有効。Wireshark 3.0+ で復号する場合は SSLKEYLOGFILE の TLS 1.3 形式 (5 種 SECRET) が必要 → [C-wireshark.md](C-wireshark.md)
- **ECH (Encrypted Client Hello, [RFC 9460](https://datatracker.ietf.org/doc/html/rfc9460))** が Chrome 117+ / Firefox 118+ / Cloudflare で段階的に有効化。ECH が成立すると outer ClientHello に public 名 (例: `cloudflare-ech.com`) しか出ず、SNI 観察ができなくなる
- **HTTP/3 (QUIC over UDP/443, [RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000))** は Chrome / Safari / Firefox でデフォルト有効。OS の HTTP proxy 設定では intercept されない → [A-mitmproxy.md](A-mitmproxy.md) の対応参照
- **Chrome 拡張内部の通信**は `chrome://extensions` → 該当拡張 → Service Worker → Inspect で DevTools が開く

「HTTPS は復号できないが SNI は見える」は 2026 年では条件付き (ECH 成立時は見えない)。
