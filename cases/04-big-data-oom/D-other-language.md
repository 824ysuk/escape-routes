# 事例 4-D: 別言語に切り替え（Go と Rust）

## まず検討すべき: 「自前で書く」前に既存実装を使う

「Rust を書く」前に、**Rust 製の [Polars](https://pola.rs/)** や **C++ 製の [DuckDB](https://duckdb.org/)** を Python / Node から呼ぶ選択肢を検討する。PyO3 / N-API 経由でネイティブ速度が出る。実務の 90% はこれで済む。

| 状況 | 推奨 |
|---|---|
| Python / Node から呼べばよい | **Polars / DuckDB を呼ぶ** ([04-A](A-stream.md), [04-C](C-sqlite-duckdb.md)) |
| 既存ライブラリで足りない | Go / Rust 自前実装 (本ファイル) |
| 配布バイナリ単体が要件 | Go (single binary) |
| 厳密なメモリ制御 / embedded | Rust |
| 定期実行 / 高 throughput / latency 重視 | Go |

## Go vs Rust の trade-off

| | Go | Rust |
|---|---|---|
| GC | あり (sub-ms target) | なし (borrow checker でコンパイル時所有権検証) |
| 学習コスト | 低 | 高 (lifetime / 借用) |
| コンパイル時間 | 速い | 遅い (数十秒〜分) |
| メモリ使用量 | やや多め (GC) | 最小 |
| ecosystem | 豊富、stdlib 充実 | 急成長中、tokio/async が分裂気味 |
| `latency` の予測可能性 | GC pause で稀にスパイク | 安定 |
| 適性 | CLI / web server / cloud | embedded / kernel / 厳密 latency |

CSV を 1 回処理するだけなら Go で十分。定常 service にするなら好み。

## 前提 / install

```bash
# Go 1.21+
brew install go
go version
mkdir csv-process && cd csv-process
go mod init example.com/csv-process

# Rust (rustup install)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cargo --version
cargo new csv-process-rs
cd csv-process-rs
# Cargo.toml の [dependencies] に csv = "1.3" を追加
```

## コード

**Go**:

```go
package main

import (
	"encoding/csv"
	"fmt"
	"io"
	"os"
)

func main() {
	f, err := os.Open("huge.csv")
	if err != nil { panic(err) }
	defer f.Close()

	r := csv.NewReader(f)
	r.FieldsPerRecord = -1
	r.LazyQuotes = true

	header, err := r.Read()
	if err != nil { panic(err) }
	fmt.Println("columns:", header)

	count := 0
	for {
		row, err := r.Read()
		if err == io.EOF { break }
		if err != nil { panic(err) }
		_ = row
		count++
	}
	fmt.Println("rows:", count)
}
```

**Rust**:

```rust
// src/main.rs — Cargo.toml に csv = "1.3" を追加
use csv::ReaderBuilder;
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let mut rdr = ReaderBuilder::new().flexible(true).from_path("huge.csv")?;
    let headers = rdr.headers()?.clone();
    println!("columns: {:?}", headers);

    let mut count = 0;
    for result in rdr.records() {
        let _record = result?;
        count += 1;
    }
    println!("rows: {}", count);
    Ok(())
}
```

```bash
# Go 実行
go run main.go
# Rust 実行
cargo run --release
```

## 期待出力

- `columns: [id amount category ...]`
- `rows: 123456789`
- メモリ使用量は数 MB 程度（`csv.Reader` / Rust `csv::Reader` は 1 行ずつ読む stream）
- `/usr/bin/time -v` で実測すると `Maximum resident set size` が 10-30 MB 程度

## ハマりポイント

- Go の `bufio.Scanner` は quoted field 内の改行を正しく扱えない → CSV では必ず `encoding/csv`
- Rust は Cargo.toml の dependency 設定で初回 build に時間がかかる (数十秒〜数分)
- どちらも release build (`go build` / `cargo build --release`) で速度が大きく変わる
- 配布バイナリにするなら Go の `CGO_ENABLED=0 go build -ldflags="-s -w"` で 5MB 前後の single static binary

### 参考

- [Go GC guide](https://go.dev/doc/gc-guide)
- [Rust book](https://doc.rust-lang.org/book/)
- [Polars (Rust 製 DataFrame、Python/Node bindings)](https://pola.rs/posts/polars_birds_eye_view/)
