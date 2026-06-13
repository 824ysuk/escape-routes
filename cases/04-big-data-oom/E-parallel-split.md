# 事例 4-E: 並列分割（map-reduce）

## 前提 / install

```bash
brew install parallel    # GNU parallel (macOS)
sudo apt install parallel # Debian / Ubuntu
parallel --version

# 初回 citation prompt を抑制 (CI で詰まる頻発バグ #1)
mkdir -p ~/.parallel && touch ~/.parallel/will-cite
```

- 各 chunk を処理する Python script `process_chunk.py` を用意
- CSV 専用なら `mlr` (Miller) / `xsv` / `csvkit` が header を保持したまま split できる

## コード

### 推奨: `parallel --pipe-part --header :` で 1 ステップ

中間ファイル不要、header 自動複製、mmap ベースで高速:

```bash
parallel --will-cite \
  --pipe-part --block 100M --header : -a huge.csv \
  python process_chunk.py > result.out
# --block 100M : 各 chunk を約 100MB ずつ
# --header :   : 1 行目を全 chunk に複製
# -a huge.csv  : input ファイル (mmap で分割)
```

### 旧方式: `split` を介する場合 (header 破壊バグ対策)

```bash
# 1. CSV の header を別ファイルに保存
head -n 1 huge.csv > header.csv

# 2. data 行のみを 10 万行ずつに split（header を破壊しないため tail で skip）
tail -n +2 huge.csv | split -l 100000 - chunk_

# 3. 各 chunk に header を再付加
for f in chunk_??; do
  cat header.csv "$f" > "$f.csv"
  rm "$f"
done

# 4. GNU parallel で 8 並列実行（失敗したら停止、merge 順を chunk 名で sort）
ls chunk_*.csv | sort | parallel --will-cite --halt now,fail=1 -j 8 \
  'python process_chunk.py {} > {.}.out'

# 5. 結果を結合（sort で順番を確定）
cat $(ls chunk_*.out | sort) > result.out

# 6. 後片付け
rm chunk_*.csv chunk_*.out header.csv
```

### CSV 専用ツール (header を持ったまま分割)

```bash
# Miller (mlr): CSV header 自動処理
mlr --csv split -n 100000 huge.csv -o chunk_

# xsv (Rust 製、最速): split で行数指定
xsv split -s 100000 chunks/ huge.csv

# csvkit: in2csv で正規化してから split
csvkit csvgrep -c amount huge.csv > clean.csv
```

`process_chunk.py`:

```python
import sys, csv
with open(sys.argv[1]) as f:
    r = csv.DictReader(f)
    total = sum(float(row['amount']) for row in r)
print(total)
```

## 期待出力

- `parallel --pipe-part` 方式: 標準出力に各 chunk の集計結果が並ぶ。`result.out` の行数 = chunk 数
- `split` 方式: `chunk_aa.csv`, `chunk_ab.csv`, ... が作成（各 10 万行 + header 1 行）、`result.out` に集計結果
- 件数照合は `duckdb -c "SELECT COUNT(*) FROM read_csv('huge.csv')"` で論理行数を取る (`wc -l` は quoted newline で誤る)

## ハマりポイント

### GNU parallel の地雷

- **`--citation` プロンプト**: 初回実行時に academic citation のプロンプトで 30 秒停止する (CI で詰まる原因 #1)。`~/.parallel/will-cite` を touch するか各コマンドに `--will-cite` を付ける
- 失敗時の停止: `--halt now,fail=1` を付けないと失敗 job が無視され結果が欠ける

### CSV header 破壊バグ

- **`split -l` は header 行を考慮しない**: `tail -n +2` + 再付加が必須。`parallel --pipe-part --header :` を使えば自動回避
- merge の順序: `cat chunk_*.out` の glob 順は環境依存 → `sort` で明示

### `split` の suffix 制約

- デフォルト suffix は 2 文字 = 最大 676 chunk。`-l 100000` なら data 約 6,760 万行が上限 → それ以上は `split -a 3 -l 100000` で suffix を 3 文字に拡張し、後続の glob (`chunk_??` / `chunk_*.csv`) も `chunk_???` に合わせる

### disk / メモリ

- `split` 方式は disk 容量を元ファイルの約 2 倍消費 → `/tmp` を事前確認
- `--pipe-part` は mmap ベースなので中間ファイル不要、disk I/O 半減

### より大きなデータ (TB+)

GNU parallel は単機 multi-process の最下層。TB+ では中間層に上げる:

- [Dask](https://docs.dask.org/): `dask.dataframe.read_csv('huge.csv', blocksize='256MB').groupby('category').sum().compute()` で out-of-core 並列
- [Ray Data](https://docs.ray.io/en/latest/data/data.html): `ray.data.read_csv()` で multi-node 並列
- Apache Beam / Spark: portable batch/stream pipeline
