# 事例 3-E: tcpdump / strace

## 前提 / install

```bash
brew install tcpdump            # macOS
sudo apt install tcpdump        # Debian / Ubuntu
brew install --cask wireshark   # GUI + tshark
sudo apt install strace         # Linux 専用
# macOS は dtruss（プリインストール）
```

- sudo 権限必須（または `setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump`）
- コンテナ: `kubectl debug` で ephemeral container を attach、または sidecar 追加

## コード

```bash
# tcpdump: 特定ポート + 特定ホスト
sudo tcpdump -i any -w /tmp/cap.pcap 'port 8080 and host upstream.example.com'

# 循環バッファ運用（10MB × 5 ファイル循環）
sudo tcpdump -i any -w /tmp/cap-%Y%m%d-%H%M%S.pcap -C 10 -W 5 -G 3600 'port 8080 and host upstream.example.com'

# pcap を tshark で要約
tshark -r /tmp/cap.pcap -Y 'http.response.code >= 500' -T fields -e ip.src -e ip.dst -e http.request.method -e http.request.uri -e http.response.code

# 実出力例:
# 10.0.1.5    10.0.2.10   POST    /api/v1/orders  503
# 10.0.1.5    10.0.2.11   GET     /api/v1/users   500

# strace: 稼働中プロセスの syscall を観察
sudo strace -f -e trace=network,openat,connect -p $(pgrep -f myapp) -o /tmp/strace.log

# macOS の dtruss
sudo dtruss -p $(pgrep -f myapp) 2>&1 | grep -E 'connect|read|write'
```

## 期待出力

- `/tmp/cap.pcap` に pcap 形式で通信が保存（数十 MB〜）
- tshark の TAB 区切り出力で HTTP 500 を発生させた src/dst/URL
- strace のログで syscall 単位の呼び出し
- 循環バッファ運用時は時間付きファイル名で saved

## ハマりポイント

- HTTPS は中身が暗号化。本文が必要なら [mitmproxy（事例 5-A）](../05-invisible-traffic/A-mitmproxy.md)
- strace の出力は大量（数 GB / 分）。`-e` で必要な syscall に絞る
- macOS では SIP の制約で `any` インターフェース取れないことあり → 個別インターフェース指定（`-i en0`）
- 長時間観察は循環バッファ必須（`-C` `-W` `-G`）
- **pcap の取り扱い**: 平文 HTTP は Authorization / Cookie / PII を全て含み、TLS でも SNI / ClientHello / 接続パターンが残る。`/tmp` 直書きは world-readable で事故源。書き出し先は `install -d -m 700 /var/tmp/pcap-$USER` で専用 dir を作り、解析後 `shred -u`（macOS は `rm -P`）で破棄。S3 等共有 storage に上げない（[OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)）
- 本番では `-s 0` 既定でディスクが即埋まる。header のみで足りるなら `-s 256` 等で payload を切り詰める（[tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html)）
- `tcpdump -i any` は Linux の cooked-mode で VLAN tag / 一部 L2 情報を取りこぼす。L2 情報が必要なら個別 IF + BPF filter（高トラフィック環境のディスク圧迫対策にもなる）
- HTTP/3 (QUIC = UDP 443) を観察対象に含めるなら `'(tcp or udp) port 443'` の filter にする。`tcp port 443` だけでは pass-through
