# 事例 2: 認証必須サービスから定期取得したい

> Status: 🚧 WIP（Notion 原稿から転記予定）

## 何が起きるか

対象サービスが OAuth 2.0 (RFC 6749) + TOTP（2 段階認証）または SAML SSO を要求する。`curl` で全フローを再現するのは、redirect 連鎖・state パラメータ・PKCE・TOTP 入力が組み合わさり実用に耐えない。

## 代替手段

| 手段 | 概要 | 再現性 |
|---|---|---|
| A. Cookie 抽出 + curl | ブラウザの認証済み Cookie を抜き出し curl で送る | session 期限まで |
| B. 公式 API + OAuth Device Flow | RFC 8628 でブラウザ操作と CLI を疎結合化 | 恒久的、refresh で延命 |
| C. Playwright で完全自動化 | TOTP は pyotp で生成 | UI 変更で壊れる |
| D. mitmproxy で観察 → 再現 | 必要な API call だけ抽出して自前で叩く | API 仕様変更で壊れる |
| E. ブラウザ拡張機能で抜く | ユーザーがブラウザを使うだけでデータ送信 | ブラウザ開いている間だけ |

## 実装例

各代替手段の詳細は近日転記。

## 補足

「OAuth を curl で再現する」は得られる物に対してコストが見合わないことが多い。session を借りる（A・C・E）か、観察して必要部分だけ再現する（D）方が短期では現実的。
