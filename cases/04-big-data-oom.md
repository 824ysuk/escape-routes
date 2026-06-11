# 事例 4: 巨大データで OOM が出る

> Status: 🚧 WIP（Notion 原稿から転記予定）

## 何が起きるか

`JSON.parse(fs.readFileSync(...))` や `pd.read_csv` でファイル全体をメモリに載せようとして OOM。ファイルサイズが物理メモリを超えていたり、データ構造のオーバーヘッドで 3-5 倍に膨らんだりする。

## 代替手段

| 手段 | 概要 | 前提 |
|---|---|---|
| A. stream / generator に書き換え | Node.js Streams / Python generator で逐次処理 | 処理が逐次で完結する |
| B. メモリ上限を上げる | Node なら `--max-old-space-size`、JVM なら `-Xmx` | 物理メモリに余裕がある |
| C. 一時 SQLite / DuckDB に書き出す | データを DB に流し込んで SQL で処理 | 処理がクエリで表現できる |
| D. 別言語に切り替え | Go / Rust など、メモリ効率の高い言語で書き直す | 定期実行する、再利用される |
| E. 並列分割（map-reduce） | チャンク化して別プロセス / マシンで処理 | 処理が独立に分割可能 |

## 実装例

各代替手段の詳細は近日転記。

## 補足

「全部メモリに載せて処理する」は手数が少なく書きやすいが、データが想定の 10 倍になった瞬間に破綻する。最初から stream / SQLite で書いておくと、後で OOM が出ても引き出しが利く。データ処理パターンの体系は Designing Data-Intensive Applications の "Batch Processing" / "Stream Processing" 章が網羅的。
