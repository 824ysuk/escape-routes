# 事例 1-D: 公式 API / データダウンロード機能

## 前提 / install

- `jq` install: `brew install jq` / `apt install jq`
- Business / Creator アカウント切り替え: Instagram アプリ → 設定 → アカウント → プロアカウントに切り替え
- Facebook ページ作成 + Instagram アカウント連携
- [Facebook Developer Console](https://developers.facebook.com/) でアプリ作成 → 「製品を追加」で Instagram Graph API → Graph API Explorer 画面（左上のドロップダウンでアプリ選択 → "User or Page" → "Get User Access Token" → スコープに `instagram_basic`, `pages_show_list`, `pages_read_engagement` → "Generate Access Token"）
- 環境変数 `ACCESS_TOKEN` / `APP_ID` / `APP_SECRET` に保存
- 個人ユーザーのデータダウンロード（API 不使用）: Instagram → 設定 → アカウントセンター → 個人の情報 → 情報のダウンロード

## コード

```bash
# 1. 自分の Instagram Business アカウント ID を取得
curl -G 'https://graph.facebook.com/v20.0/me/accounts' \
  --data-urlencode "access_token=$ACCESS_TOKEN" \
  -o pages.json
IG_USER_ID=$(jq -r '.data[0].instagram_business_account.id' pages.json)
echo "ig_user_id=$IG_USER_ID"

# 2. メディア一覧（pagination 対応: next を辿る）
NEXT="https://graph.facebook.com/v20.0/${IG_USER_ID}/media?fields=id,caption,media_url,media_type,timestamp,permalink&access_token=${ACCESS_TOKEN}&limit=100"
> media.ndjson
while [ -n "$NEXT" ] && [ "$NEXT" != "null" ]; do
  curl -s "$NEXT" -o page.json
  jq -c '.data[]' page.json >> media.ndjson
  NEXT=$(jq -r '.paging.next // empty' page.json)
done

# 3. 短期 token (1h) を長期 token (60d) に交換
curl -G 'https://graph.facebook.com/v20.0/oauth/access_token' \
  --data-urlencode 'grant_type=fb_exchange_token' \
  --data-urlencode "client_id=$APP_ID" \
  --data-urlencode "client_secret=$APP_SECRET" \
  --data-urlencode "fb_exchange_token=$ACCESS_TOKEN" \
  -o long_token.json
LONG_TOKEN=$(jq -r .access_token long_token.json)
```

## 期待出力

- `pages.json` に Facebook Pages 一覧、`instagram_business_account.id` が ig-user-id
- `media.ndjson` に投稿が 1 行 1 件の NDJSON で蓄積。1 行例:

```json
{"id":"17841...","caption":"...","media_url":"https://scontent...","media_type":"IMAGE","timestamp":"2026-06-01T00:00:00+0000","permalink":"https://www.instagram.com/p/..."}
```

- `long_token.json` に `{"access_token":"...","token_type":"bearer","expires_in":5183944}`（60 日 ≒ 5,184,000 秒）
- データダウンロード機能: 数時間〜数日後にメール通知 → アカウントセンターから ZIP DL（中身は `media/posts_1.json` 等の JSON + 画像 / 動画ファイル）

## ハマりポイント

- 短期 token は 1 時間で失効。長期 token への交換が運用上必須
- Instagram Graph API は Business / Creator 限定。個人アカウントは「データダウンロード」機能のみ
- レート制限: 1 時間あたり 200 calls / user
- 開発モード（App Review 未通過）では追加した「テスター」ユーザーのみアクセス可。公開アプリ化には App Review が必要
