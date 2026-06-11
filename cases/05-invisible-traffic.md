# 事例 5: ブラウザに見えない通信を観察したい

> Status: 🚧 WIP（Notion 原稿から転記予定）

## 何が起きるか

ブラウザの DevTools Network タブで観察できるのは、そのブラウザが発した通信だけ。以下は見えない: モバイルアプリの通信、デスクトップアプリの通信、別端末からの通信、ブラウザ拡張内部の通信、暗号化されていて中身が読めない通信。

## 代替手段

| 手段 | 概要 | 用途 |
|---|---|---|
| A. mitmproxy | ローカル proxy として動作、HTTPS を自前 CA で復号 | HTTPS API の観察、編集、再送 |
| B. Charles Proxy / Proxyman | GUI 版の HTTPS proxy（有料） | GUI で目視確認したい |
| C. Wireshark / tshark | 生のパケットを観察。HTTPS は復号できないが SNI・TCP・TLS handshake は見える | プロトコル層のトラブル、低レベル観察 |
| D. iOS Web Inspector / Android Studio Network Profiler | 公式の OS / IDE ツール。CA インストール不要で安全 | 自社モバイルアプリのデバッグ |
| E. アプリにデバッグログを仕込む | リクエスト直前にログを print | 1 リクエストだけ確認したい |

## 実装例

各代替手段の詳細は近日転記。

## 補足

DevTools で見える「ブラウザの通信」は氷山の一角。プロトコル層を 1 段下げる（Wireshark）か、proxy を挟む（mitmproxy）か、アプリの中に入る（E）かで観察可能な領域が広がる。
