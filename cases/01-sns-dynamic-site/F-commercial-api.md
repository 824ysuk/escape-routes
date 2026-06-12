# 事例 1-F: 商用 scraping API

## 前提 / install

- [ScrapingBee](https://www.scrapingbee.com/) で sign up（無料 1,000 credit、有料 $49/月〜）
- API key 取得 → 環境変数 `API_KEY` に保存
- 対象サイトの `robots.txt` を確認、利用規約も併せて確認
- 環境変数 `TARGET_URL` に取得対象 URL を設定

## コード

```bash
# 標準（JS 実行込み、約 5 credit / call）
curl -G 'https://app.scrapingbee.com/api/v1/' \
  --data-urlencode "api_key=$API_KEY" \
  --data-urlencode "url=$TARGET_URL" \
  --data-urlencode 'render_js=true' \
  -o rendered.html -w 'status=%{http_code}\n'

# 高難易度サイト用（住宅 IP、約 25 credit / call）
curl -G 'https://app.scrapingbee.com/api/v1/' \
  --data-urlencode "api_key=$API_KEY" \
  --data-urlencode "url=$TARGET_URL" \
  --data-urlencode 'render_js=true' \
  --data-urlencode 'premium_proxy=true' \
  --data-urlencode 'country_code=jp' \
  -o rendered.html
```

## 期待出力

- 標準出力: `status=200`
- `rendered.html`（数百 KB）に JS 実行後の完全な HTML
- レスポンス header `Spb-Cost` に消費 credit、`Spb-Resolved-Url` に最終 URL、`Spb-Request-Id` で問い合わせ用 ID

## ハマりポイント

- `render_js=true` = 5 credit、`premium_proxy=true` = 25 credit
- 失敗（status=4xx, 5xx）時は credit を消費しない（refund される）
- 対象サービスの ToS 違反になる場合あり: 必ず `robots.txt` と利用規約を確認。法的責任は利用者
- レスポンスが空 / 壊れる場合は `js_scenario` で待機戦略を追加
