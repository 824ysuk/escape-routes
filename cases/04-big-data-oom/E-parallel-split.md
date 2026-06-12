# 事例 4-E: 並列分割（map-reduce）

## 前提 / install

```bash
brew install parallel    # GNU parallel (macOS)
sudo apt install parallel # Debian / Ubuntu
parallel --version
```

- 各 chunk を処理する Python script `process_chunk.py` を用意

## コード

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
ls chunk_*.csv | sort | parallel --halt now,fail=1 -j 8 'python process_chunk.py {} > {.}.out'

# 5. 結果を結合（sort で順番を確定）
cat $(ls chunk_*.out | sort) > result.out

# 6. 後片付け
rm chunk_*.csv chunk_*.out header.csv
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

- `chunk_aa.csv`, `chunk_ab.csv`, ... が作成（各 10 万行 + header 1 行）
- `result.out` に各 chunk の集計結果（行数 = chunk 数）: `1234567.89` / `2345678.90` ...
- `wc -l huge.csv` と `cat chunk_*.csv | wc -l` の data 行数が一致

## ハマりポイント

- **`split -l` は header 行を考慮しない**: 上のように `tail -n +2` + 再付加が必須
- 失敗時の停止: `--halt now,fail=1` を付けないと失敗 job が無視され結果が欠ける
- merge の順序: `cat chunk_*.out` の glob 順は環境依存 → `sort` で明示
- disk 容量: 元ファイル + 各 chunk + header の合計でほぼ 2 倍消費 → `/tmp` の容量を事前確認
- **`split` のデフォルト suffix は 2 文字 = 最大 676 chunk**: 超えると `split: too many files` で停止する。`-l 100000` なら data 約 6,760 万行が上限 → それ以上は `split -a 3 -l 100000` で suffix を 3 文字に拡張し、後続の glob（`chunk_??` / `chunk_*.csv`）も `chunk_???` に合わせる
