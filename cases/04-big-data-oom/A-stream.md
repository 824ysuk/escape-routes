# 事例 4-A: stream / generator に書き換え

## 前提 / install

**Node.js 版**:

```bash
node --version   # v18+ を確認
npm install csv-parse csv-stringify
# package.json に "type": "module" を追加するか、ファイル拡張子を .mjs にする
```

**Python 版**:

```bash
python --version   # 3.8+
pip install pandas   # chunksize オプションを使う場合
```

**JSON 向け追加**:

```bash
# Node.js
npm install stream-json stream-chain

# Python
pip install ijson
```

## コード

**Node.js**:

```javascript
// stream-convert.mjs
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { parse } from 'csv-parse';
import { stringify } from 'csv-stringify';
import { Transform } from 'node:stream';

await pipeline(
  createReadStream('huge.csv'),
  parse({ columns: true, skip_empty_lines: true }),
  new Transform({
    objectMode: true,
    transform(row, _enc, cb) {
      cb(null, { id: row.id, amount: Number(row.amount) * 1.1 });
    }
  }),
  stringify({ header: true }),
  createWriteStream('out.csv')
);
console.log('done');
```

**Python generator**:

```python
# stream_convert.py — 標準ライブラリのみで stream 処理
import csv

with open('huge.csv', newline='') as fin, open('out.csv', 'w', newline='') as fout:
    reader = csv.DictReader(fin)
    writer = csv.DictWriter(fout, fieldnames=['id', 'amount'])
    writer.writeheader()
    for row in reader:
        writer.writerow({'id': row['id'], 'amount': float(row['amount']) * 1.1})

# pandas の場合は chunksize で逐次処理
# import pandas as pd
# for chunk in pd.read_csv('huge.csv', chunksize=10_000):
#     chunk['amount'] = chunk['amount'] * 1.1
#     chunk.to_csv('out.csv', mode='a', header=False, index=False)
```

**Node.js（JSON 配列、stream-json）**:

```javascript
// stream-json.mjs — 巨大 JSON 配列を逐次処理
import { createReadStream } from 'node:fs';
import { chain } from 'stream-chain';
import { parser } from 'stream-json';
import StreamArray from 'stream-json/streamers/StreamArray.js';

const pipeline = chain([
  createReadStream('huge.json'),
  parser(),
  StreamArray.withParser(),
]);

let count = 0;
pipeline.on('data', ({ value }) => {
  // value は配列内の各オブジェクト
  count += 1;
});
pipeline.on('end', () => console.log(`processed ${count} items`));
```

**Python（JSON 配列、ijson）**:

```python
# stream_json.py — ijson で大きな JSON 配列を逐次処理
import ijson

with open('huge.json', 'rb') as f:
    # JSON 配列のルート要素を 1 つずつ取り出す（'item' は配列要素のパス）
    for item in ijson.items(f, 'item'):
        # item は dict
        pass
```

## 期待出力

- `out.csv` に変換後 CSV が保存される
- `wc -l huge.csv` と `wc -l out.csv` が同数（empty line を除く）であれば件数照合 OK
- メモリ使用量を実測:

```bash
# Linux
/usr/bin/time -v node stream-convert.mjs 2>&1 | grep 'Maximum resident'
# 結果例: Maximum resident set size (kbytes): 65432   (約 64 MB)
# macOS
/usr/bin/time -l node stream-convert.mjs 2>&1 | grep 'maximum resident'
```

## ハマりポイント

- backpressure: 消費側（write）が遅いと内部バッファが溜まる → `highWaterMark` を `16` から `64` 程度（object mode の場合、object 数）に調整
- header が無い CSV は `columns: false` にして row を配列で扱う
- ESM / CJS 互換: `package.json` の `"type": "module"` か `.mjs` 拡張子で ESM 化
- pandas の `chunksize` は内部で全件読まないため OOM を回避できる
- 巨大 JSON は構造による: `[{...}, {...}, ...]` 形式は stream-json / ijson で逐次化、改行区切り（JSONL = 1 行 1 JSON）なら Node.js は `readline`、Python は `for line in f: json.loads(line)` で済む
