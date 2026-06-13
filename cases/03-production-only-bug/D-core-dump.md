# 事例 3-D: core dump + デバッガ

> **対象範囲**: 自社が運用する production binary に限る。core dump にはメモリ全域が含まれるため、credential / PII が含まれる前提で扱う。

## 前提 / install

```bash
brew install gdb        # macOS (codesign 必要)
sudo apt install gdb    # Linux
go install github.com/go-delve/delve/cmd/dlv@latest
```

### Linux で core dump を取る (modern distro 配慮)

Ubuntu 16.04+ / RHEL 8+ / Debian 11+ は `core_pattern` を **systemd-coredump** (`|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h`) または **apport** (`|/usr/share/apport/apport`) で pipe handler に置き換え済。`tee` で上書きすると distro の crash 解析機能が壊れ、package upgrade で揺り戻しも起きる。

**手順は以下の順序で**:

1. 現状確認 (上書きする前に必ず):
   ```bash
   cat /proc/sys/kernel/core_pattern
   ```
2. systemd-coredump が動いている場合は `coredumpctl` で取得:
   ```bash
   coredumpctl list                    # 最近の crash 一覧
   coredumpctl debug <PID>              # gdb 直結
   coredumpctl dump <PID> -o core.dump  # ファイルとして書き出し
   ```
3. apport の場合: `/var/lib/apport/coredump/` を確認
4. どうしても plain file が必要な場合は **distro の hook を残して別パスへ**:
   ```bash
   # systemd-tmpfiles 経由で /var/coredumps を確保
   echo 'd /var/coredumps 0755 root root -' | sudo tee /etc/tmpfiles.d/coredumps.conf
   sudo systemd-tmpfiles --create
   # distro hook を保ったまま使えるか確認 (coredumpctl で取れる)
   ```

### サービスごとの core 許可

**systemd 配下** (modern Linux server の 99%) — `limits.conf` は pam_limits 経由 (login session 時のみ評価) で systemd unit には届かない:

```ini
# /etc/systemd/system/myapp.service.d/coredump.conf
[Service]
LimitCORE=infinity
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
systemctl show myapp -p LimitCORE   # infinity を確認
```

**interactive shell のみ** (補助):
```bash
ulimit -c unlimited
# /etc/security/limits.conf に永続化は login session 用、systemd 配下では効かない
```

### Container で core を取る (Docker / K8s)

`core_pattern` は **host namespace の値のみ有効** で container 内 sysctl では変更不可。

```bash
# 1. host で
echo '/var/coredumps/core.%e.%p.%t' | sudo tee /proc/sys/kernel/core_pattern

# 2. container 起動 (Docker)
docker run --ulimit core=-1 \
  -v /var/coredumps:/var/coredumps \
  myimage:latest

# 3. K8s の場合 DaemonSet で hostPath マウント
# spec:
#   securityContext:
#     allowPrivilegeEscalation: false
#   containers:
#   - name: app
#     resources: { limits: { memory: 2Gi } }
#     volumeMounts:
#     - { name: coredumps, mountPath: /var/coredumps }
#   volumes:
#   - { name: coredumps, hostPath: { path: /var/coredumps } }
```

`%P` (namespace-aware PID) と `%p` (host PID) を区別する。container 内 OOM kill は cgroup memory.max が rlimit より小さい場合に発生し core が出ない。

### macOS

`/cores/` 配下に出力される (`/proc/sys/...` は存在しない):

```bash
sudo sysctl -w kern.coredump=1
sudo sysctl -w kern.corefile=/cores/core.%P
ls /cores/
```

### 対象 binary の debug 情報戦略

production binary に debug info を残すか別配信するか:

| 言語 | 推奨 | 理由 |
|---|---|---|
| C/C++ | strip + 別 `.debug` artifact + [debuginfod](https://sourceware.org/elfutils/Debuginfod.html) | binary size を抑えつつ symbol を CI から push |
| Go | full symbol を埋めたまま (10-20% 増、許容範囲) | `dlv core` が DWARF を binary から読む。**`-N -l` は production で外す** (inline/最適化 OFF で 30% 悪化) |
| Rust release | `[profile.release] debug = "line-tables-only"` | size と解析性能のバランス |

BuildID (`/usr/lib/debug/.build-id/<hash>/`) で binary と symbol を一意に紐付ける。

## コード

```bash
# 1. core を gdb で開く (Linux、stripped binary + 別 .debug を組み合わせる場合)
gdb /path/to/binary /var/coredumps/core.myapp.1234

# (gdb プロンプト内で)
# set follow-fork-mode child         # gunicorn の preforked worker 解析
# set print thread-events off         # 出力ノイズ削減
# info threads                        # thread 数把握
# bt full                             # 全フレームの backtrace + 変数
# thread apply all bt full            # 全 thread + locals 一括
# frame 3                             # 該当フレームに移動
# print *request                      # ポインタの中身

# CPython を解析する場合 (libpython 拡張)
gdb -ex 'add-auto-load-safe-path /usr/share/gdb/auto-load' \
    -ex 'set follow-fork-mode child' \
    -ex 'thread apply all bt full' \
    -ex 'py-bt' \
    -ex 'py-locals' \
    --batch /usr/bin/python3 core.1234
# py-bt / py-list / py-locals は /usr/share/gdb/auto-load/usr/bin/python*-gdb.py から提供

# macOS の lldb
lldb /path/to/binary -c /cores/core.1234
# thread backtrace all
# frame select 3
# frame variable
# p *request

# Go の delve (release binary + GOTRACEBACK=crash で再現できる)
dlv core /path/to/binary /var/coredumps/core.myapp.1234
# goroutines
# goroutine 1
# bt
# locals
```

### Go: release binary のまま core を取る

```bash
# production binary を 再 build せずに core を取りたい場合
GOTRACEBACK=crash ./myapp
# SIGABRT 時に core dump する (default は "single" で core を出さない)

# runtime からの制御
# import "runtime/debug"
# debug.SetCrashOutput(file, debug.CrashOptions{})
```

`-N -l` (inline / 最適化 OFF) は production binary を 30% 程度悪化させる可能性があり、症状を消すこともある。**canary node に限定して試す**。

## 期待出力

- `bt full` で crash 時点の関数 call chain と各フレームのローカル変数
- 例: `#0  0x00007f... in segfault_handler () at handler.c:42`
- stripped binary だと `??()` ばかり並ぶ → debuginfod / 別 `.debug` 配信が必要

## ハマりポイント

- stripped binary では変数名・行番号が出ない → debuginfod / `objcopy --add-gnu-debuglink` で symbol を後付け
- macOS の `gdb` は codesign が必要 → `lldb` を使う方が楽
- core dump はサイズが大きい (数 GB〜) → disk full に注意。`/var/coredumps` の容量を事前確保
- core dump には **メモリ全域** が含まれる (credential / PII 込み)。共有先・保管期間を厳重に
- 参考: [systemd-coredump(8) man page](https://man7.org/linux/man-pages/man8/systemd-coredump.8.html) / [Linux core_pattern docs](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/kernel.html#core-pattern) / [Python devguide: gdb](https://devguide.python.org/development-tools/gdb/) / [Go runtime env vars](https://pkg.go.dev/runtime#hdr-Environment_Variables)
