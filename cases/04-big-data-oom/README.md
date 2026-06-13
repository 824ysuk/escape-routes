# 事例 4: 巨大データで OOM が出る

## 何が起きるか

`JSON.parse(fs.readFileSync(...))` や `pd.read_csv` でファイル全体をメモリに載せようとして OOM。ファイルサイズが物理メモリを超えていたり、データ構造のオーバーヘッドで 3-5 倍に膨らんだりする。

## 決定フロー (データ形状ベース)

「手段表」より先に、データの形状と目的で分岐を考えると現代的な選択になる。

```
何をしたい?
├─ 集計 / JOIN / GROUP BY → DuckDB (C) ── 10-100x 速い
├─ 1 行ずつ変換 → stream (A: Polars LazyFrame / Node Streams / Python generator)
├─ 一時的にメモリを上げて時間稼ぎ → メモリ拡張 (B: --max-old-space-size / MaxRAMPercentage)
├─ TB 級 / 水平分割 → 並列分割 (E: parallel --pipe-part / Dask / Ray)
├─ 配布バイナリ単体が要件 / メモリ厳密 → Go / Rust 自前 (D)
└─ ファイル形式を選べる → Parquet に変換して DuckDB で読む (A 末尾参照)
```

「集計か変換か」「1 回か定期か」「GB か TB か」で分岐する。多くのケースは A (stream / Polars / DuckDB) と C (DuckDB) の組み合わせで済む。

## 代替手段

| 手段 | 概要 | 前提 |
|---|---|---|
| [A. stream / generator に書き換え](A-stream.md) | Node.js Streams / Python generator / **Polars LazyFrame / DuckDB / PyArrow streaming** | 処理が逐次で完結する |
| [B. メモリ上限を上げる](B-memory-limit.md) | Node `--max-old-space-size`、JVM `-XX:MaxRAMPercentage`、cgroup limit 確認 | 物理メモリに余裕がある |
| [C. 一時 SQLite / DuckDB に書き出す](C-sqlite-duckdb.md) | データを DB に流し込んで SQL で処理。**OLAP は DuckDB、OLTP は SQLite** | 処理がクエリで表現できる |
| [D. 別言語に切り替え](D-other-language.md) | Go / Rust など。**まず Polars/DuckDB を呼ぶ選択肢を検討** | 定期実行する、再利用される、配布バイナリ単体が要件 |
| [E. 並列分割（map-reduce）](E-parallel-split.md) | `parallel --pipe-part` / Dask / Ray / Spark | 処理が独立に分割可能 |

## 他手段を選ぶ条件

- **B（メモリ拡張）**: 暫定対応で時間を稼ぎたい、ファイルがあと数倍で済む見込み
- **C（SQLite / DuckDB）**: 集計・JOIN・複雑なフィルタが必要、コードより SQL の方が短く済む。集計中心なら DuckDB が公式ベンチで pandas / SQLite 比で 10-100x の性能差を示す ([DuckDB benchmarks](https://duckdblabs.github.io/db-benchmark/))
- **D（別言語）**: 定期実行で書き換え投資が回収できる、または依存ライブラリが他言語にしかない。配布バイナリ単体が要件
- **E（並列分割）**: ファイルが本当に大きい (TB 級)、横方向に水平分割できる

## 補足

「全部メモリに載せて処理する」は手数が少なく書きやすいが、データが想定の 10 倍になった瞬間に破綻する。最初から stream / SQLite / DuckDB で書いておくと、後で OOM が出ても引き出しが利く。

データ処理パターンの体系は [Designing Data-Intensive Applications](https://dataintensive.net/) の "Batch Processing" / "Stream Processing" 章が網羅的。Arrow ecosystem (Polars / DuckDB / PyArrow) の全体観は [Apache Arrow docs](https://arrow.apache.org/docs/) を参照。
