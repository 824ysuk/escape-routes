# 事例 4-C: 一時 SQLite / DuckDB に書き出す

## SQLite vs DuckDB の使い分け

| | SQLite | DuckDB |
|---|---|---|
| データモデル | row-oriented | column-oriented + vectorized execution |
| 用途 | OLTP (point lookup, transaction) | OLAP (集計, JOIN, 分析) |
| 集計速度 (GROUP BY SUM 等) | 普通 | **10-100x 速い** |
| transaction / UPDATE 中心 | 強い | 弱い |
| Parquet / S3 / HTTP 直接読み込み | × | ✅ |
| install | プリインストール多い | install 必要 |

本ケースのような **GROUP BY SUM はすべて DuckDB が圧倒的に速い**。OLTP (1 行更新が頻発) なら SQLite。

## 前提 / install

```bash
# macOS
brew install sqlite3
brew install duckdb

# Debian / Ubuntu
sudo apt install sqlite3
curl https://install.duckdb.org | sh   # DuckDB 公式 install script (https://duckdb.org/install/)

# 確認
sqlite3 --version
duckdb --version
```

## コード

### DuckDB (集計には第一選択)

```bash
# CSV を直接 SQL で参照 (型推論あり、中間 table 不要、v1.0+ では read_csv が canonical)
duckdb -c "SELECT category, SUM(amount) AS total FROM read_csv('huge.csv') GROUP BY category ORDER BY total DESC LIMIT 10"

# memory_limit と threads を制御 (default は自動だが、共存環境では明示)
duckdb -c "SET memory_limit='4GB'; SET threads=4;
  SELECT category, SUM(amount) FROM read_csv('huge.csv') GROUP BY category"

# Parquet も直接読める (column pruning + 圧縮で 10-100x 軽い)
duckdb -c "SELECT category, SUM(amount) FROM read_parquet('huge.parquet') GROUP BY category"

# S3 / HTTP 上のファイルを直接 query (httpfs extension)
duckdb -c "INSTALL httpfs; LOAD httpfs;
  SELECT * FROM read_parquet('s3://bucket/data/*.parquet') LIMIT 100"

# 結果を CSV / Parquet に書き出し
duckdb -c "COPY (SELECT * FROM read_csv('huge.csv') WHERE category='A') TO 'out.parquet' (FORMAT PARQUET)"
```

### SQLite (OLTP 中心の場合)

```bash
# CSV の header を確認
head -1 huge.csv
# 例: id,amount,category,timestamp

# 巨大 CSV を import するときは事前 PRAGMA で 10-100x 速くする
sqlite3 data.db <<'EOF'
PRAGMA journal_mode = WAL;          -- WAL モードで concurrency 改善
PRAGMA synchronous = NORMAL;        -- fsync 緩和 (PERSISTENT より速い)
PRAGMA cache_size = -2000000;       -- 約 2GB cache (負値は KB 単位)
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 30000000000;     -- mmap で 30GB まで

BEGIN TRANSACTION;
.mode csv
.headers on
.import huge.csv records
COMMIT;

CREATE INDEX idx_category ON records(category);
SELECT category, SUM(CAST(amount AS REAL)) AS total
  FROM records GROUP BY category ORDER BY total DESC LIMIT 10;
EOF
```

## 期待出力

- 標準出力に集計結果 (最大 10 行)
- DuckDB の場合: 1 行目に header `category,total`、続いて `category_a,123456789.0` / `category_b,98765432.1` ...
- SQLite 経由の場合 `data.db` が残る (CSV と同程度のサイズ)

## ハマりポイント

### DuckDB API の更新

- v0.10 (2024-02) 以降 `read_csv_auto()` は `read_csv()` に統合され auto_detect がデフォルト `true`。**v1.0+ では `read_csv('huge.csv')` が canonical** (`read_csv_auto` は alias として残る)
- 同様に `read_json_auto` → `read_json`、`read_parquet` は元から auto detect

### SQLite の罠

- **SQLite `.import` は全カラムを TEXT として保存**: 数値比較が文字列比較になり結果が狂う → `CAST(amount AS REAL)` で明示
- `.import` の header skip 挙動は version 依存。SQLite 3.32+ の `.import --csv huge.csv records` は **空 table への import 時のみ** 1 行目を header として列名に使う。既存 table への追記時は header もデータとして入る。事前に table を作らない運用にするか、`tail -n +2 huge.csv | sqlite3 ...` で header を落として import する方が確実 ([SQLite CLI: CSV Import](https://www.sqlite.org/cli.html#csv_import))
- `.import` を `BEGIN; ... COMMIT;` で囲まないと毎行 fsync で 10-100x 遅い
- 型推論・header 検知・quote 内 newline の総合的な堅さでは DuckDB の `read_csv` が優位。新規導入なら DuckDB を第一選択にする

### DuckDB の memory / threads

- default で全 core 使用、メモリも自動で 80% 確保するが、共存環境では `SET memory_limit='4GB'; SET threads=4;` で制御
- container では cgroup memory limit を超えないよう明示推奨

### 参考

- [DuckDB CSV docs](https://duckdb.org/docs/data/csv/overview.html)
- [DuckDB httpfs extension](https://duckdb.org/docs/extensions/httpfs/overview.html)
- [SQLite PRAGMA reference](https://www.sqlite.org/pragma.html)
- [DuckDB 1.0 release announcement (2024-06)](https://duckdb.org/2024/06/03/announcing-duckdb-100.html)
