# 事例 2: 認証必須サービスから定期取得したい

## 何が起きるか

対象サービスが OAuth 2.0 (RFC 6749) + TOTP（2 段階認証）または SAML SSO を要求する。`curl` で全フローを再現するのは、redirect 連鎖・state パラメータ・PKCE・TOTP 入力が組み合わさり実用に耐えない。

## 代替手段

| 手段 | 概要 | 再現性 |
|---|---|---|
| [A. Cookie 抽出 + curl](A-cookie-curl.md) | ブラウザの認証済み Cookie を抜き出し curl で送る | session 期限まで |
| [B. 公式 API + OAuth Device Flow](B-device-flow.md) | RFC 8628 でブラウザ操作と CLI を疎結合化 | 恒久的、refresh で延命 |
| [C. Playwright で完全自動化](C-playwright.md) | TOTP は pyotp で生成 | UI 変更で壊れる |
| [D. mitmproxy で観察 → 再現](D-mitmproxy.md) | 必要な API call だけ抽出して自前で叩く | API 仕様変更で壊れる |
| [E. ブラウザ拡張機能で抜く](E-extension.md) | ユーザーがブラウザを使うだけでデータ送信 | ブラウザ開いている間だけ |

## 他手段を選ぶ条件

- **B（Device Flow）**: 公式 API が用意されている、長期運用したい
- **C（Playwright）**: UI を経由する必要がある（API がない）、完全自動化したい
- **D（mitmproxy 観察）**: API は内部に存在するが公式ドキュメントが無い、軽量に書きたい
- **E（拡張）**: ユーザーが普段から使うサービスから、副次的にデータを取りたい

## 補足

「OAuth を curl で再現する」は得られる物に対してコストが見合わないことが多い。session を借りる（A・C・E）か、観察して必要部分だけ再現する（D）方が短期では現実的。
