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
  // URL を別ファイル urls.txt に書き出し、shell 側は xargs で読む。
  // URL に "" / $() / ` が混入してもシェル展開されない経路にする。
  const urlsTxt = urls.join('\n') + '\n';
  const script = [
    '#!/bin/bash',
    'set -euo pipefail',
    'mkdir -p downloaded && cd downloaded',
    "i=0; while IFS= read -r u; do",
    '  i=$((i+1))',
    '  # Content-Type に従って拡張子を決める (image/* 以外は .bin 化)',
    "  curl -# --remote-header-name -fL -o \"$(printf '%03d.bin' $i)\" \"$u\" || echo \"fail: $u\" >&2",
    'done < ../urls.txt',
  ].join('\n');
  const dl = (blob, name) => {
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = name;
    a.click();
  };
  dl(new Blob([urlsTxt], { type: 'text/plain' }), 'urls.txt');
  dl(new Blob([script], { type: 'text/plain' }), 'download.sh');
})();
```

## 期待出力

- Console に `42 件の URL を検出` のような件数表示
- Downloads フォルダに `urls.txt` と `download.sh`（数 KB〜数十 KB）が保存される
- `urls.txt` と `download.sh` を同じ directory に置いて `bash ./download.sh` を実行 → `downloaded/` 配下に `001.bin`, `002.bin`, ... と連番で保存
- 各ファイルで curl の進捗バー（`#####`）が表示される

## ハマりポイント

- CDN URL の `oe=` パラメータは hex 表記の Unix epoch second (例: `oe=6859ABCD` → `printf '%d\n' 0x6859ABCD` で UTC 秒)。Instagram は配信から 1〜2 時間で失効。生成後は即時 `bash` 実行
- 認可は `oe=` (有効期限) + `oh=` (hmac) のペア。**IP / referrer 縛りは無い** ので、ブラウザで取得した URL を別マシンから curl しても期限内なら取れる
- Safari は Blob URL の自動ダウンロードに制約あり → 失敗時は `console.log(urls.join('\n'))` で URL を手動コピー → ターミナルで `curl -O`
- iOS Safari の DevTools は macOS Safari からの接続が必要（端末単独不可）
- `<picture>` / `<source>` を使うサイトでは `<img>` だけ拾うと取りこぼし。上のコードは両方考慮
- セキュリティ警告: 生成された `download.sh` は実行前に必ず内容を目視確認する。URL を文字列展開で `curl ".."` に埋め込む書き方は、URL 中に `$()` / `` ` `` / `"` が含まれた場合に任意コード実行へつながる。本コードでは URL を別ファイル `urls.txt` に書き出して `while read` で読み込ませることで、シェル展開を経由しない経路にしている（参考: [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)）
- 拡張子を `.jpg` 固定にすると video / webp / heic が混じる場合に Content-Type と不一致になる。実体に合わせる場合は `curl --remote-header-name` + サーバの `Content-Disposition` か、別途 `file(1)` で判定する
