# 事例 1: SNS / 動的サイトからデータが取れない

> **対象範囲**: 取得対象は利用者自身のアカウント、または公開アカウントの公開情報に限る。第三者の非公開コンテンツ・大規模収集は Meta / X / TikTok の ToS 違反 + 不正競争防止法。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 何が起きるか

`curl` や HTTP ライブラリで GET すると 403 が返る、あるいは HTML は取れるが投稿・画像のデータが入っていない（テンプレートだけ）。Instagram・X・LinkedIn・TikTok 等が該当する。原因は（1）サーバーサイドのボット検知（User-Agent・IP・JA3 fingerprint 等）、（2）JavaScript で投稿データを後から読み込む動的レンダリング、の 2 つ。

## 代替手段

| 手段 | 概要 | 再現性 |
|---|---|---|
| [A. ブラウザのコンソールで JS を実行](A-browser-console.md) | DOM + Performance API からメモリ取り出し | 一度きり向け |
| [B. Cookie + curl + UA 偽装](B-cookie-curl.md) | ブラウザ Cookie 抽出 → curl で送る | session 期限まで |
| [C. 特化 OSS](C-specialized-oss.md) | yt-dlp / gallery-dl / instaloader | 定期実行可 |
| [D. 公式 API / データダウンロード機能](D-official-api.md) | Instagram Graph API、ユーザー設定の DL 機能 | 恒久的 |
| [E. ヘッドレスブラウザ + stealth](E-headless-stealth.md) | Playwright + playwright-stealth | 定期自動化向け |
| [F. 商用 scraping API](F-commercial-api.md) | ScrapingBee / Bright Data | 大量・継続向け（有料） |

## 他手段を選ぶ条件

- **B（Cookie + curl）**: 同じデータを別日に再取得したい、スクリプト化したい
- **C（特化 OSS）**: そのサービスが OSS でカバーされている、CLI 一発で済ませたい
- **D（公式）**: 自社 / 自分のアカウントで、定期業務として組み込みたい
- **E（Playwright stealth）**: ログインや無限スクロール等の操作が必要
- **F（商用 API）**: 大量・継続的、自前運用したくない

## 補足

A が最速で動くのは「アカウントは自分」「1 回きりで良い」「ブラウザが既に開いていた」状況。定期実行が必要なら C か D、操作が複雑なら E、コストで殴れるなら F が合う。
