# 事例 1-A: ブラウザのコンソールで JS を実行

## 前提 / install

- install 不要
- 推奨ブラウザ: Chrome 90+ / Firefox 88+ / Safari 14+
- DevTools を開く: macOS `⌘ + ⌥ + I`、Windows / Linux `F12`、Safari は 設定 → 詳細 → 「メニューバーに開発メニューを表示」を ON 後 `⌘ + ⌥ + C`
- CSP block 時は Console に `Refused to create a worker from 'blob:...'` が出る → E（Playwright）にフォールバック

## コード

```javascript
(function () {
  const CDN = 'cdninstagram.com';  // X = twimg.com、TikTok = tiktokcdn.com に置換
  const urls = [...new Set([
    ...[...document.querySelectorAll('img, source')]
      .map(el => el.src || el.srcset?.split(',')[0]?.trim().split(' ')[0])
      .filter(s => s && s.includes(CDN)),
    ...performance.getEntriesByType('resource')
      .filter(r => r.name.includes(CDN) && r.initiatorType === 'img')
      .map(r => r.name)
  ])];
  console.log(`${urls.length} 件の URL を検出`);
  if (!urls.length) { console.warn('URL が見つかりません'); return; }
  const script = [
    '#!/bin/bash',
    'mkdir -p downloaded && cd downloaded',
    ...urls.map((u, i) => `curl -# -o "${String(i + 1).padStart(3, '0')}.jpg" "${u}"`)
  ].join('\n');
  const a = document.createElement('a');
  a.href = URL.createObjectURL(new Blob([script], { type: 'text/plain' }));
  a.download = 'download.sh';
  a.click();
})();
```

## 期待出力

- Console に `42 件の URL を検出` のような件数表示
- Downloads フォルダに `download.sh`（数 KB〜数十 KB）が保存される
- `bash ~/Downloads/download.sh` を実行 → `downloaded/` 配下に `001.jpg`, `002.jpg`, ... と連番で画像保存
- 各ファイルで curl の進捗バー（`#####`）が表示される

## ハマりポイント

- CDN URL の `oe=` パラメータ（UNIX タイムスタンプ）は配信の有効期限。Instagram は 1〜2 時間程度。生成後は即時 `bash` 実行
- Safari は Blob URL の自動ダウンロードに制約あり → 失敗時は `console.log(urls.join('\n'))` で URL を手動コピー → ターミナルで `curl -O`
- iOS Safari の DevTools は macOS Safari からの接続が必要（端末単独不可）
- `<picture>` / `<source>` を使うサイトでは `<img>` だけ拾うと取りこぼし。上のコードは両方考慮
