# 事例 4-A: stream / generator に書き換え

## 前提 / install

**Node.js 版**:

```bash
node --version   # v20+ 推奨 (v22 LTS で動作確認、v18 は 2025-04 EOL)
npm install csv-parse csv-stringify
# package.json に "type": "module" を追加するか、ファイル拡張子を .mjs にする
```

**Python 版**:

```bash
python --version   # 3.8+
pip install pandas    # chunksize 用 (歴史的選択肢)
pip install polars    # 現代的な代替: LazyFrame + streaming engine
pip install duckdb    # SQL で集計・変換するなら最速
pip install pyarrow   # zero-copy Arrow streaming
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
  // columns 明示で Transform 出力の object key 順に依存させない (csv-stringify 公式推奨)
  stringify({ header: true, columns: ['id', 'amount'] }),
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

# pandas の chunksize は歴史的選択肢
# 最初の chunk だけ header=True で書き、以降は append (i == 0 で切り替える)
# import pandas as pd
# for i, chunk in enumerate(pd.read_csv('huge.csv', chunksize=10_000)):
#     chunk['amount'] = chunk['amount'] * 1.1
#     chunk.to_csv('out.csv', mode='w' if i == 0 else 'a', header=(i == 0), index=False)
```

**Python (Polars LazyFrame — 現代的)**:

```python
# stream_polars.py — Polars の streaming engine で OOM 回避 + ベクトル化で 10-100x 速い
import polars as pl

(
    pl.scan_csv('huge.csv')                          # LazyFrame として開く (まだ読まない)
      .with_columns((pl.col('amount') * 1.1).alias('amount'))
      .select(['id', 'amount'])
      .sink_csv('out.csv')                           # streaming 実行
)
```

**Python (DuckDB — SQL で集計が短く済むなら最速)**:

```python
import duckdb
duckdb.sql("""
    COPY (
        SELECT id, amount * 1.1 AS amount
        FROM read_csv('huge.csv')
    ) TO 'out.csv' (FORMAT CSV, HEADER)
""")
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

**JSONL (1 行 1 JSON、ndjson)** — **大規模データの第一選択**:

JSON 配列 `[{...}, {...}]` は最後の `]` を待たないと parse できず stream に向かない。JSONL (`{...}\n{...}\n`) なら 1 行 = 1 record で `readline` / `for line` で逐次処理できる。生成側は `jq -c '.[]'` で配列 → JSONL に変換可能。

Node.js:

```javascript
// jsonl-stream.mjs
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

const rl = createInterface({
  input: createReadStream('huge.jsonl'),
  crlfDelay: Infinity,   // \r\n / \n 両方を 1 改行として扱う
});

let count = 0;
for await (const line of rl) {
  if (!line.trim()) continue;   // 空行 skip
  const obj = JSON.parse(line);
  count += 1;
  // process(obj);
}
console.log(`processed ${count} items`);
```

Python:

```python
# jsonl_stream.py
import json

with open('huge.jsonl') as f:
    count = 0
    for line in f:
        line = line.strip()
        if not line:
            continue
        obj = json.loads(line)
        count += 1
        # process(obj)
print(f'processed {count} items')
```

JSON 配列を JSONL に変換するワンライナー:

```bash
jq -c '.[]' huge.json > huge.jsonl
# 逆変換: jq -s '.' huge.jsonl > huge.json
```

## 期待出力

- `out.csv` に変換後 CSV が保存される
- 件数照合は `wc -l` ではなく論理行数を取る (quoted newline で物理行 ≠ 論理行):
  ```bash
  duckdb -c "SELECT COUNT(*) FROM read_csv('huge.csv')"
  duckdb -c "SELECT COUNT(*) FROM read_csv('out.csv')"
  ```
- メモリ使用量を実測:

```bash
# Linux
/usr/bin/time -v node stream-convert.mjs 2>&1 | grep 'Maximum resident'
# 結果例: Maximum resident set size (kbytes): 65432   (約 64 MB)
# macOS
/usr/bin/time -l node stream-convert.mjs 2>&1 | grep 'maximum resident'
```

## ハマりポイント

### backpressure

- `pipeline()` を使えば backpressure は自動で処理される (`writable.write()` が `false` を返すと内部で pause、`drain` で resume)
- `highWaterMark` を上げるのは backpressure 解決ではなくバッファ容量増加で、消費側が遅いとメモリ消費が膨らみ OOM リスクが上がる。先に slow consumer 側の最適化 (DB の bulk insert 化、批処理) を検討する

### データ形式

- 巨大 JSON は構造による: `[{...}, {...}, ...]` 形式は stream-json / ijson で逐次化、改行区切り (JSONL) なら標準ライブラリで済む (本文の「JSONL」小節参照)
- データ形式を選べる場合は **Parquet** が最優先 (column pruning + 圧縮で 10-100x 軽い)。`pyarrow.parquet.ParquetFile('x.parquet').iter_batches(batch_size=10000)` で 1 column だけ読める

### Python の選択指針

- **新規実装は Polars LazyFrame / DuckDB を推奨**。pandas の chunksize は手書きで concat する必要があり、Arrow ベース 2 ツールにメモリ効率で劣る
- 「pandas の API を維持しつつ stream」が要件なら [Dask](https://docs.dask.org/) が中間解 (`dask.dataframe.read_csv('huge.csv', blocksize='256MB')`)

### その他

- header が無い CSV は `columns: false` にして row を配列で扱う
- ESM / CJS 互換: `package.json` の `"type": "module"` か `.mjs` 拡張子で ESM 化
- Node.js v22+ は `Readable.toWeb()` / `Readable.fromWeb()` で Web Streams API (fetch の `response.body` 等) と相互運用可能
