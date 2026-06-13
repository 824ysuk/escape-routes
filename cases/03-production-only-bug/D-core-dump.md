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

# Ubuntu 20.04+ / Debian 12+ / RHEL 8+ は kernel.core_pattern が
#   |/lib/systemd/systemd-coredump %P %u %g %s %t %c %h
# になっており、systemd-coredump 経由で journald 集約される。
# まずはこちらの標準経路で取得を試す方が安全 (file 上書きで他サービスの
# core 集約が壊れない)。
coredumpctl list                       # 既存 core 一覧
coredumpctl gdb <PID|exe>              # 該当 core を gdb で開く
coredumpctl dump <PID> > /tmp/core.bin # file に書き出したいとき

# どうしても /tmp/core-* file 形式に切り替えるなら、元 pattern を必ず退避してから
ORIG_PATTERN=$(cat /proc/sys/kernel/core_pattern)
echo '/tmp/core-%e.%p' | sudo tee /proc/sys/kernel/core_pattern
# 復元 (作業終了後):
# echo "$ORIG_PATTERN" | sudo tee /proc/sys/kernel/core_pattern

# 永続化は systemd-coredump.conf 経由 (Storage=external 等) を推奨。
# /etc/sysctl.d/ で core_pattern を直接書き換えると distro 既定の
# systemd-coredump 集約が無効化される副作用がある。
sudo systemctl edit --full systemd-coredump@.service   # 必要に応じて Storage 等を調整
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
- 現代 distro は `core_pattern` を `|systemd-coredump` / `|apport` の pipe 形式で初期化している。file 出力に上書きすると (a) journald 集約が無効化、(b) 同居サービスの core が `/tmp` に溢れる、(c) 永続化 conf を残すと再起動後も副作用が続く。安全な手順は **(1) `coredumpctl` 経由を試す → (2) どうしても file 出力したいときだけ元 pattern を退避してから書き換え → (3) 作業終了後に必ず復元**（[systemd-coredump(8)](https://man7.org/linux/man-pages/man8/systemd-coredump.8.html) / [coredump.conf(5)](https://man7.org/linux/man-pages/man5/coredump.conf.5.html)）
