# 事例 1-C: 特化 OSS（yt-dlp / gallery-dl / instaloader）

## 前提 / install

```bash
python --version   # 3.8+
pip install -U yt-dlp gallery-dl instaloader
brew install ffmpeg   # yt-dlp の動画/音声 merge 用
```

## コード

```bash
# yt-dlp: 単発投稿（動画・写真両対応）
yt-dlp \
  --sleep-interval 2 --max-sleep-interval 5 \
  -o '%(upload_date)s_%(id)s.%(ext)s' \
  'https://www.instagram.com/p/POST_SHORTCODE/'

# 認証付き（login wall 越え）
yt-dlp --cookies-from-browser chrome 'https://www.instagram.com/p/POST_SHORTCODE/'

# gallery-dl: アカウント全投稿、フォルダ自動整理
gallery-dl --sleep 2 -d ./downloads 'https://www.instagram.com/USERNAME/'
gallery-dl --cookies-from-browser firefox 'https://www.instagram.com/USERNAME/'

# instaloader: プロフィール / ストーリーズ / ハイライト
instaloader profile USERNAME --no-videos --no-metadata-json
instaloader --login YOUR_LOGIN_USERNAME stories
```

## 期待出力

- yt-dlp: カレント directory に `20260601_POST_ID.mp4` 等で保存。stderr に `[download] 50.0% of 12.34MiB at 2.1MiB/s` 等
- gallery-dl: `./downloads/instagram/USERNAME/` 配下に投稿が画像順で保存。stderr に `[instagram][info] image 1/30`
- instaloader: `./USERNAME/` ディレクトリ作成、`2026-06-11_12-00-00_UTC.jpg` 形式で保存

## ハマりポイント

### shadowban / soft ban の発火条件と回復

同 cookie で 1 時間以内に 200 req 超 → 24-72h の soft ban で 401 / `Login required` / `checkpoint_required` が返る。

**シグナル**:

- `HTTP 429 Too Many Requests` 連続
- JSON `{"message":"checkpoint_required"}` → アプリ側で本人確認必須
- redirect to `/accounts/login/`
- 取得した profile JSON で `is_private: true` が増える

**回復手順**:

1. 24-72h 完全停止
2. Instagram アプリで本人確認・パスワード変更
3. cookie を取り直し
4. 別 IP (住宅 proxy or 別 ISP) で再開

**予防**:

- yt-dlp: `--sleep-interval 2 --max-sleep-interval 5` (本ファイルの例で既に対応)
- instaloader: `--request-timeout 300` + `RateController` subclass で 30-90 秒 jitter
- 同時並列度: `--max-connections 1`

### バージョン管理

- 各 OSS は Instagram 仕様変更で壊れる → `pip install -U` で必ず最新化
- 2024 以降 [Meta が JA3/JA4 を導入](https://scrapfly.io/blog/how-to-avoid-web-scraping-blocking-tls/)し、内部で [curl_cffi](https://github.com/yifeikong/curl_cffi) (BoringSSL 経由) に切り替えている fork / 案件がある

### Cookie の罠

- `--cookies-from-browser` 使用時、macOS Keychain への access 認証 prompt が出ることがある
- Chrome 起動中だと Cookie DB が file lock される → Chrome を完全終了してから実行
- 最小必須 cookie (Instagram): `sessionid`, `csrftoken`, `ds_user_id`, `mid` (詳細は [B-cookie-curl.md](B-cookie-curl.md) 参照)

### その他

- instaloader のストーリーズは `--login` 必須
- ffmpeg 未 install だと yt-dlp が動画 + 音声 merge に失敗（音声付き動画は別ファイルで残る）

### 参考

- [instaloader troubleshooting (Login error)](https://instaloader.github.io/troubleshooting.html#login-error)
- [yt-dlp issue #8226 (Instagram rate limit)](https://github.com/yt-dlp/yt-dlp/issues/8226)
