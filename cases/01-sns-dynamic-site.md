# 事例 1: SNS / 動的サイトからデータが取れない

> Status: 🚧 WIP（Notion 原稿から転記予定）

## 何が起きるか

`curl` や HTTP ライブラリで GET すると 403 が返る、あるいは HTML は取れるが投稿・画像のデータが入っていない（テンプレートだけ）。Instagram・X・LinkedIn・TikTok 等が該当する。原因は（1）サーバーサイドのボット検知（User-Agent・IP・JA3 fingerprint 等）、（2）JavaScript で投稿データを後から読み込む動的レンダリング、の 2 つ。

## 代替手段

| 手段 | 概要 | 再現性 |
|---|---|---|
| A. ブラウザのコンソールで JS を実行 | DOM + Performance API からメモリ取り出し | 一度きり向け |
| B. Cookie + curl + UA 偽装 | ブラウザ Cookie 抽出 → curl で送る | session 期限まで |
| C. 特化 OSS | yt-dlp / gallery-dl / instaloader | 定期実行可 |
| D. 公式 API / データダウンロード機能 | Instagram Graph API、ユーザー設定の DL 機能 | 恒久的 |
| E. ヘッドレスブラウザ + stealth | Playwright + playwright-stealth | 定期自動化向け |
| F. 商用 scraping API | ScrapingBee / Bright Data | 大量・継続向け（有料） |

## 実装例

各代替手段の「前提 / install → コード → 期待出力 → ハマりポイント」の詳細は近日転記。

## 補足

A が最速で動くのは「アカウントは自分」「1 回きりで良い」「ブラウザが既に開いていた」状況。定期実行が必要なら C か D、操作が複雑なら E、コストで殴れるなら F が合う。
