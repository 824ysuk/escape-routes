# 事例 4: 巨大データで OOM が出る

> Status: 🚧 WIP（各代替手段の詳細は `.notion-source/04-big-data-oom.md` から個別ファイルへ転記中）

## 何が起きるか

`JSON.parse(fs.readFileSync(...))` や `pd.read_csv` でファイル全体をメモリに載せようとして OOM。ファイルサイズが物理メモリを超えていたり、データ構造のオーバーヘッドで 3-5 倍に膨らんだりする。

## 代替手段

| 手段 | 概要 | 前提 |
|---|---|---|
| [A. stream / generator に書き換え](A-stream.md) | Node.js Streams / Python generator で逐次処理 | 処理が逐次で完結する |
| [B. メモリ上限を上げる](B-memory-limit.md) | Node なら `--max-old-space-size`、JVM なら `-Xmx` | 物理メモリに余裕がある |
| [C. 一時 SQLite / DuckDB に書き出す](C-sqlite-duckdb.md) | データを DB に流し込んで SQL で処理 | 処理がクエリで表現できる |
| [D. 別言語に切り替え](D-other-language.md) | Go / Rust など、メモリ効率の高い言語で書き直す | 定期実行する、再利用される |
| [E. 並列分割（map-reduce）](E-parallel-split.md) | チャンク化して別プロセス / マシンで処理 | 処理が独立に分割可能 |

## 他手段を選ぶ条件

- **B（メモリ拡張）**: 暫定対応で時間を稼ぎたい、ファイルがあと数倍で済む見込み
- **C（SQLite / DuckDB）**: 集計・JOIN・複雑なフィルタが必要、コードより SQL の方が短く済む
- **D（別言語）**: 定期実行で書き換え投資が回収できる、または依存ライブラリが他言語にしかない
- **E（並列分割）**: ファイルが本当に大きい（TB 級）、横方向に水平分割できる

## 補足

「全部メモリに載せて処理する」は手数が少なく書きやすいが、データが想定の 10 倍になった瞬間に破綻する。最初から stream / SQLite で書いておくと、後で OOM が出ても引き出しが利く。データ処理パターンの体系は Designing Data-Intensive Applications の "Batch Processing" / "Stream Processing" 章が網羅的。
