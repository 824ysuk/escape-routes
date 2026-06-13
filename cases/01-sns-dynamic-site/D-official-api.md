# 事例 1-D: 公式 API / データダウンロード機能

> **対象範囲**: Instagram Graph API は自分の Business / Creator アカウント、または同連携の Facebook ページに紐づくデータに限る。任意の第三者アカウントの内容は取れない (下記マトリクス参照)。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 取れる / 取れないマトリクス (Instagram Graph API)

| 対象 | 可否 | 経路 |
|---|---|---|
| **自分の Business / Creator media** | ✅ 取れる | `/{ig-user-id}/media` |
| **自分の Business / Creator insights** | ✅ 取れる | `/{media-id}/insights` |
| **他人の Business / Creator media** (username 既知) | △ 一部 | [Business Discovery API](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/business_discovery)、profile + 最近の media |
| **他人の Personal アカウントの media** | ✗ 取れない | 公式 API は存在しない |
| **任意 hashtag の最近の投稿** | △ 制限あり | [`ig_hashtag_search` → `recent_media`](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-hashtag/recent-media): 過去 24h、30 件/週 hashtag 検索上限 |
| **Stories** | ✅ 自分のみ | `instagram_manage_insights` 権限 + App Review 必須 |
| **comments** | ✅ 自分の media のみ | `/{media-id}/comments` |
| **DM** | △ Business のみ | [Messenger Platform / Instagram Messaging API](https://developers.facebook.com/docs/messenger-platform/instagram/) |

「Graph API で何でも取れる」は誤り。第三者の個人投稿は公式経路では取れない。

## 前提 / install

- `jq` install: `brew install jq` / `apt install jq`
- Business / Creator アカウント切り替え: Instagram アプリ → 設定 → アカウント → プロアカウントに切り替え
- Facebook ページ作成 + Instagram アカウント連携
- [Facebook Developer Console](https://developers.facebook.com/) でアプリ作成 → 「製品を追加」で Instagram Graph API → Graph API Explorer 画面（左上のドロップダウンでアプリ選択 → "User or Page" → "Get User Access Token" → スコープに `instagram_basic`, `pages_show_list`, `pages_read_engagement` → "Generate Access Token"）
- 環境変数 `ACCESS_TOKEN` / `APP_ID` / `APP_SECRET` に保存。`umask 077` で新規ファイルを 600 に
- 個人ユーザーのデータダウンロード (API 不使用): Instagram → 設定 → アカウントセンター → 個人の情報 → 情報のダウンロード

```bash
# 機密値が history に残らないように
set +o history
umask 077

# Graph API バージョンを環境変数化 (約 2 年で deprecate されるため)
GRAPH_API_VERSION="${GRAPH_API_VERSION:-v23.0}"
```

## コード

```bash
# 1. 自分の Instagram Business アカウント ID を取得
curl -G "https://graph.facebook.com/${GRAPH_API_VERSION}/me/accounts" \
  --data-urlencode "access_token=$ACCESS_TOKEN" \
  -o pages.json
IG_USER_ID=$(jq -r '.data[0].instagram_business_account.id' pages.json)
echo "ig_user_id=$IG_USER_ID"

# 2. メディア一覧（pagination 対応: next を辿る）
NEXT="https://graph.facebook.com/${GRAPH_API_VERSION}/${IG_USER_ID}/media?fields=id,caption,media_url,media_type,timestamp,permalink&access_token=${ACCESS_TOKEN}&limit=100"
> media.ndjson
while [ -n "$NEXT" ] && [ "$NEXT" != "null" ]; do
  curl -s "$NEXT" -o page.json
  jq -c '.data[]' page.json >> media.ndjson
  NEXT=$(jq -r '.paging.next // empty' page.json)
done

# 3. 短期 token (1h) を長期 token (60d) に交換
curl -G "https://graph.facebook.com/${GRAPH_API_VERSION}/oauth/access_token" \
  --data-urlencode 'grant_type=fb_exchange_token' \
  --data-urlencode "client_id=$APP_ID" \
  --data-urlencode "client_secret=$APP_SECRET" \
  --data-urlencode "fb_exchange_token=$ACCESS_TOKEN" \
  -o long_token.json
LONG_TOKEN=$(jq -r .access_token long_token.json)

# 4. 60 日前に rolling refresh する
curl -G "https://graph.instagram.com/refresh_access_token" \
  --data-urlencode 'grant_type=ig_refresh_token' \
  --data-urlencode "access_token=$LONG_TOKEN" \
  -o long_token-new.json

# 5. revoke (権限失効) — ユーザーの「Business Integrations」と同じ効果
curl -X DELETE "https://graph.facebook.com/${GRAPH_API_VERSION}/me/permissions" \
  -d "access_token=$LONG_TOKEN"
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

### Token のライフサイクル

- 短期 token は 1 時間で失効。長期 token への交換が運用上必須
- 長期 token は 60 日で expire し、**rolling refresh** (`grant_type=ig_refresh_token`) が必要
- 漏洩時は `/me/permissions` の DELETE で revoke ([RFC 7009 token revocation](https://datatracker.ietf.org/doc/html/rfc7009) 概念)
- `long_token.json` は `.gitignore` 対象 (CONTRIBUTING.md の Secret hygiene 参照)、`chmod 600` 必須

### API バージョン管理

- Graph API バージョンは概ね四半期更新、各バージョンは **約 2 年で deprecate → 即時 403** ([Changelog](https://developers.facebook.com/docs/graph-api/changelog/))
- ハードコードで放置すると数ヶ月〜2 年で全コードが死ぬ。`GRAPH_API_VERSION` を環境変数化し、四半期ごとに Changelog を確認 → 互換性試験

### 制約と権限

- Instagram Graph API は Business / Creator 限定。個人アカウントは「データダウンロード」機能のみ
- レート制限: 1 時間あたり 200 calls / user。Business Discovery API は別枠で 200/h、hashtag search は別 quota
- 開発モード（App Review 未通過）では追加した「テスター」ユーザーのみアクセス可。公開アプリ化には App Review が必要
