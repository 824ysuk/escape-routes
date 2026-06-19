# 事例 2: 認証必須サービスから定期取得したい

> **対象範囲**: 対象は利用者自身のアカウント、または自社が運用するサービスに限る。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 何が起きるか

対象サービスが OAuth 2.0 (RFC 6749) + TOTP（2 段階認証）または SAML SSO を要求する。`curl` で全フローを再現するのは、redirect 連鎖・state パラメータ・PKCE・TOTP 入力が組み合わさり実用に耐えない。

## 代替手段

| 手段 | 概要 | 再現性 | 正当性 |
|---|---|---|---|
| [A. Cookie 抽出 + curl](A-cookie-curl.md) | ブラウザの認証済み Cookie を抜き出し curl で送る | session 期限まで | 利用者本人の session 流用 (provider 許諾なし、ToS 確認要) |
| [B. 公式 API + OAuth Device Flow](B-device-flow.md) | RFC 8628 でブラウザ操作と CLI を疎結合化 | 恒久的、refresh で延命 | OAuth 公式 grant type (RFC 8628、provider 許諾済) |
| [C. Playwright で完全自動化](C-playwright.md) | TOTP は pyotp で生成 | UI 変更で壊れる | 利用者本人の認証自動入力 (provider 許諾なし、ToS で禁止の例あり) |
| [D. mitmproxy で観察 → 再現](D-mitmproxy.md) | 必要な API call だけ抽出して自前で叩く | API 仕様変更で壊れる | 自分の端末・自分の通信のみ (ToS 観察 OK でも自動化 NG の例あり) |
| [E. ブラウザ拡張機能で抜く](E-extension.md) | ユーザーがブラウザを使うだけでデータ送信 | ブラウザ開いている間だけ | 利用者本人 (Web Store 公開時は permission justification 要) |
| [F. WebAuthn Virtual Authenticator](F-webauthn-virtual-authenticator.md) | W3C WebAuthn L3 §11 の WebDriver Extension で authenticator を仮想化、ceremony を自動化 | テスト目的・自社/自己アカウント限定 | 利用者本人 / 自社サービスのテスト自動化 (ToS 確認要) |

## 他手段を選ぶ条件

- **B（Device Flow）**: 唯一の provider 公式許諾経路。公式 API が用意されている、長期運用したい
- **A / C / E（利用者本人スコープ）**: provider 公式許諾はないが利用者自身のデータ。A は最短距離、C は UI 経由が必要・完全自動化したい、E は普段の操作から副次的に収集したい
- **D（mitmproxy 観察）**: API は内部に存在するが公式ドキュメントが無い、軽量に書きたい。観察自体は自分の通信のみだが ToS で自動化が禁止されている例あり
- **F（WebAuthn Virtual Authenticator）**: 対象サービスが passkey-only / WebAuthn 必須（TOTP は通用しない）。C は TOTP 担当、F は WebAuthn ceremony 自体を仮想化する。ToS が自動化テストを禁じていないことを確認すること

## 補足

「OAuth を curl で再現する」は得られる物に対してコストが見合わないことが多い。session を借りる（A・C・E）か、観察して必要部分だけ再現する（D）方が短期では現実的。
