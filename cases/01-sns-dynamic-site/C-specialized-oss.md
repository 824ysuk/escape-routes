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

- 各 OSS は Instagram 仕様変更で壊れる → `pip install -U` で必ず最新化
- `--cookies-from-browser` 使用時、macOS Keychain への access 認証 prompt が出ることがある
- Chrome 起動中だと Cookie DB が file lock される → Chrome を完全終了してから実行
- instaloader のストーリーズは `--login` 必須
- ffmpeg 未 install だと yt-dlp が動画 + 音声 merge に失敗（音声付き動画は別ファイルで残る）
