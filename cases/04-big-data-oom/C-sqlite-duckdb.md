# 事例 4-C: 一時 SQLite / DuckDB に書き出す

## 前提 / install

```bash
# macOS
brew install sqlite3
brew install duckdb

# Debian / Ubuntu
sudo apt install sqlite3
curl https://install.duckdb.org | sh   # DuckDB 公式 install script（https://duckdb.org/install/）

# 確認
sqlite3 --version
duckdb --version
```

## コード

```bash
# CSV の header を確認
head -1 huge.csv
# 例: id,amount,category,timestamp

# SQLite: CSV を import して SQL で集計（.import は全カラム TEXT として保存）
sqlite3 data.db <<'EOF'
.mode csv
.headers on
.import huge.csv records
CREATE INDEX idx_category ON records(category);
SELECT category, SUM(CAST(amount AS REAL)) AS total FROM records GROUP BY category ORDER BY total DESC LIMIT 10;
EOF

# DuckDB: CSV を直接 SQL で参照（型推論あり、中間 table 不要）
duckdb -c "SELECT category, SUM(amount) AS total FROM read_csv_auto('huge.csv') GROUP BY category ORDER BY total DESC LIMIT 10"
```

## 期待出力

- 標準出力に集計結果（最大 10 行）: 1 行目に header `category,total`、続いて `category_a,123456789.0` / `category_b,98765432.1` ...（`.mode csv` 設定のため `,` 区切り）
- SQLite 経由の場合 `data.db` が残る（CSV と同程度のサイズ）

## ハマりポイント

- **SQLite `.import` は全カラムを TEXT として保存**: 数値比較が文字列比較になり結果が狂う → `CAST(amount AS REAL)` で明示
- `.import` の header skip 挙動は version 依存。SQLite 3.32+ の `.import --csv huge.csv records` は **空 table への import 時のみ** 1 行目を header として列名に使う。既存 table への追記時は header もデータとして入る。事前に table を作らない運用にするか、`tail -n +2 huge.csv | sqlite3 ...` で header を落として import する方が確実（[SQLite CLI: CSV Import](https://www.sqlite.org/cli.html#csv_import)）
- 型推論・header 検知・quote 内 newline・引用符 escape の総合的な堅さでは DuckDB の `read_csv_auto` が優位。新規導入なら DuckDB を第一選択にする
- SQLite `.import` は CSV の quote 内 newline を扱えない場合あり → DuckDB を使う
