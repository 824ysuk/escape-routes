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

## 観察を制約する現代プロトコル

2024-2026 にかけて、以下 3 つが「直接観察」を制約する側になっている。手段を選ぶ前にこれらの該当条件を確認する。

| 制約 | 影響 | 該当する事象 / 対象 | 回避策 |
|---|---|---|---|
| **ECH (Encrypted Client Hello、[RFC 9460](https://datatracker.ietf.org/doc/html/rfc9460))** | outer ClientHello の SNI が暗号化されて見えない、proxy が分岐不可 | Chrome 117+ / Firefox 118+ / Cloudflare 配下 | DNS 階層で対象 host を resolve / Chrome の `chrome://flags/#encrypted-client-hello` を OFF |
| **HTTP/3 (QUIC over UDP 443、[RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000))** | OS の HTTP proxy 設定では intercept されない、tcpdump の `tcp port 443` filter で素通り | Google / Meta / Cloudflare の Alt-Svc 経由通信 | (a) Chrome の `chrome://flags#enable-quic` OFF、(b) アプリ側で Alt-Svc 無視、(c) mitmproxy 11+ の QUIC reverse mode、(d) capture filter を `(tcp or udp) port 443` に |
| **Certificate Pinning** | アプリが root CA store を見ず、特定証明書のみ受け入れる | 銀行 / 金融 / Big Tech モバイルアプリ (Twitter / Spotify 等) | (a) 自社アプリなら `network_security_config.xml` debug build / (b) Frida + Objection (自社 / 認可 pentest 限定、EULA / DMCA / 不正競争防止法 2 条 1 項 10 号への抵触に注意) |

詳細は各代替手段ファイルの該当 section ([A](A-mitmproxy.md), [B](B-charles-proxyman.md), [C](C-wireshark.md), [D](D-mobile-devtools.md)) を参照。

## 他手段を選ぶ条件

- **B（Charles / Proxyman）**: CLI が苦手、GUI で目視したい、有料ライセンスが許容できる
- **C（Wireshark）**: HTTPS の中身ではなく、コネクション失敗・TLS handshake error を見たい
- **D（公式 dev tool）**: 自社アプリで、CA インストールしたくない
- **E（デバッグログ）**: 1 リクエストだけ見たい、ソースが手元にある

## 補足

DevTools で見える「ブラウザの通信」は氷山の一角。プロトコル層を 1 段下げる（Wireshark）か、proxy を挟む（mitmproxy）か、アプリの中に入る（E）かで観察可能な領域が広がる。
