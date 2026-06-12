# 事例 3-D: core dump + デバッガ

## 前提 / install

```bash
brew install gdb        # macOS（codesign 必要）
sudo apt install gdb    # Linux
go install github.com/go-delve/delve/cmd/dlv@latest
```

- core dump 出力を許可（**Linux**）:

```bash
ulimit -c unlimited            # shell session 限定
echo '/tmp/core-%e.%p' | sudo tee /proc/sys/kernel/core_pattern
# 永続化
echo 'kernel.core_pattern = /tmp/core-%e.%p' | sudo tee /etc/sysctl.d/50-coredump.conf
echo '* soft core unlimited' | sudo tee -a /etc/security/limits.conf
echo '* hard core unlimited' | sudo tee -a /etc/security/limits.conf
```

- **macOS** の core は `/cores/` 配下に出力される（`/proc/sys/...` は存在しない）

```bash
sudo sysctl -w kern.coredump=1
sudo sysctl -w kern.corefile=/cores/core.%P
ls /cores/
```

- 対象 binary を **debug symbol 付き** で build:
    - gcc / clang: `-g -O0`
    - Go: `go build -gcflags="all=-N -l"`
    - Rust release: `[profile.release] debug = true` を Cargo.toml に
- コンテナ環境: host の `core_pattern` を設定 → host のボリュームマウント経由で core を取り出す

## コード

```bash
# 1. crash 後、core を gdb で開く（Linux）
gdb /path/to/binary /tmp/core-myapp.1234
# (gdb プロンプト内で)
# bt full              # 全フレームの backtrace + 変数
# frame 3              # 該当フレームに移動
# info locals          # ローカル変数を表示
# print *request       # ポインタの中身
# thread apply all bt  # 全 thread の stack

# macOS の lldb（gdb 不要）
lldb /path/to/binary -c /cores/core.1234
# (lldb プロンプト内で)
# thread backtrace all
# frame select 3
# frame variable
# p *request

# Go の delve
dlv core /path/to/binary /tmp/core-myapp.1234
# (dlv プロンプト内で)
# goroutines
# goroutine 1
# bt
# locals
```

## 期待出力

- `bt full` で crash 時点の関数 call chain と各フレームのローカル変数
- 例: `#0  0x00007f... in segfault_handler () at handler.c:42`
- stripped binary だと `??()` ばかり並ぶ → 失敗

## ハマりポイント

- stripped binary では変数名・行番号が出ない → `-g` 付きビルドを別途用意 or `objcopy --add-gnu-debuglink` で symbol を後付け
- macOS の `gdb` は codesign が必要 → `lldb` を使う方が楽
- core dump はサイズが大きい（数 GB〜）→ disk full に注意
- `ulimit -c unlimited` は **shell session 限定**。systemd 配下のサービスは `LimitCORE=infinity` を unit file に追加
