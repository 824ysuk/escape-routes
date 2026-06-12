# 事例 4-D: 別言語に切り替え（Go と Rust）

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
- Rust は Cargo.toml の dependency 設定で初回 build に時間がかかる（数十秒〜数分）
- どちらも release build (`go build` / `cargo build --release`) で速度が大きく変わる
