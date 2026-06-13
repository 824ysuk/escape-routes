# 事例 5-C: Wireshark / tshark

> **対象範囲**: 自分の端末・自社が運用する network に限る。共用 LAN / オフィスネットワークでの promiscuous mode capture は他人の通信傍受 (電気通信事業法 4 条) に該当しうる。

## 前提 / install

```bash
brew install --cask wireshark   # macOS (Wireshark 4.0+ 推奨)
sudo apt install wireshark tshark   # Linux

# パケットキャプチャ権限 (sudo 不要にする)
sudo dseditgroup -o edit -a $USER -t user access_bpf   # macOS
sudo usermod -aG wireshark $USER                       # Linux (再ログイン要)
```

- 自分のブラウザの TLS 通信を Wireshark で復号したい場合は環境変数 `SSLKEYLOGFILE` を設定 → Chrome / Firefox を起動 → Wireshark の Preferences → Protocols → TLS で同じファイルを指定:

```bash
export SSLKEYLOGFILE=$HOME/.ssl-key.log
chmod 600 "$SSLKEYLOGFILE" 2>/dev/null || true
open -a "Google Chrome"      # macOS
# Wireshark → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename に $SSLKEYLOGFILE を設定
```

## コード

```bash
# GUI: 特定インターフェース + capture filter (BPF) で起動
sudo wireshark -i en0 -k -f 'host api.example.com and tcp port 443'

# CLI 版 tshark: TLS handshake から接続先 SNI を列挙
# 注: ECH が成立すると SNI は public name に置換される (下記ハマりポイント参照)
sudo tshark -i any -f 'tcp port 443' -Y 'tls.handshake.type == 1' \
  -T fields -e tls.handshake.extensions_server_name

# 実出力例:
# api.example.com
# cdn.example.com

# 失敗パターン抽出 (RST = 接続リセット)
sudo tshark -i any -Y 'tcp.flags.reset == 1' \
  -T fields -e ip.src -e ip.dst -e tcp.dstport

# pcap として保存 → 後で GUI で読む
sudo tshark -i any -f 'host api.example.com' -w /tmp/cap.pcap
```

## 期待出力

- GUI: パケット一覧（時刻 / src / dst / protocol / info）+ 詳細ペイン
- tshark: 1 行 1 record の TAB 区切り出力（SNI / IP / port など）
- `SSLKEYLOGFILE` 経由で Chrome の TLS を復号した場合: Wireshark の「Decrypted TLS」ペインで HTTPS body が見える

## ハマりポイント

### TLS 1.3 と SSLKEYLOGFILE

- Wireshark **3.0+** が必要。TLS 1.3 ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)) の復号には [NSS Key Log Format](https://firefox-source-docs.mozilla.org/security/nss/legacy/key_log_format/index.html) 上の **5 種** の secret が必要
- TLS 1.2 形式: `CLIENT_RANDOM <hex> <master_secret>`
- TLS 1.3 形式: `CLIENT_HANDSHAKE_TRAFFIC_SECRET`, `SERVER_HANDSHAKE_TRAFFIC_SECRET`, `CLIENT_TRAFFIC_SECRET_0`, `SERVER_TRAFFIC_SECRET_0`, `EXPORTER_SECRET`
- Chrome 79+ / Firefox 70+ は TLS 1.3 が既定なので、Wireshark 2.x では復号できない

### capture filter (BPF) と display filter の混同

| 段階 | 文法 | 例 | 取れる field |
|---|---|---|---|
| capture filter (`-f`) | [BPF (libpcap)](https://www.tcpdump.org/manpages/pcap-filter.7.html) | `'host api.example.com and tcp port 443'` | L2-L4 と payload offset 指定のみ |
| display filter (`-Y`) | [Wireshark syntax](https://wiki.wireshark.org/DisplayFilters) | `'tls.handshake.type == 1'` | dissector 経由で L7 (`tls.*`, `http.*`) も |

capture 段階で `tls.handshake.type == 1` のような L7 field は使えない (文法エラー)。capture filter は緩く、display filter で絞り込む。

### ECH (Encrypted Client Hello) で SNI が見えなくなる

- [ECH (RFC 9460)](https://datatracker.ietf.org/doc/html/rfc9460) が成立すると ClientHello の SNI 拡張は inner ClientHello に隠蔽される
- outer ClientHello には public 名 (例: `cloudflare-ech.com`) しか出ない
- Chrome 117+ / Firefox 118+ / Cloudflare で段階的に有効化済
- `tls.handshake.extensions_server_name` が常に取れる前提は 2026 年では条件付き

### macOS の `-i any` 制約

- macOS は元々 `-i any` を持たない (BSD 設計起因、SIP は無関係)
- libpcap 1.10+ / Wireshark 4.0+ で **pktap** 経由の代替: `-i pktap,en0,en1` で複数物理 IF を束ねる
- `-i pktap,all` で全 IF capture も可能

### その他

- HTTPS の body は鍵がなければ復号不可能 → アプリのソースが手元になければ `SSLKEYLOGFILE` 経由しか道なし
- 大量パケットの pcap は GUI フィルタが遅い。CLI で先に絞ってから GUI で開く
- `SSLKEYLOGFILE` は機密情報 → 公開リポジトリに含めない、使用後に削除、`chmod 600`
- HTTP/3 (QUIC over UDP/443) は capture できるが、Wireshark の dissector は対応しつつあるところ ([RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000))

### 参考

- [Wireshark Wiki: TLS](https://wiki.wireshark.org/TLS)
- [Wireshark Wiki: CaptureFilters](https://wiki.wireshark.org/CaptureFilters) / [DisplayFilters](https://wiki.wireshark.org/DisplayFilters)
- [NSS Key Log Format docs](https://firefox-source-docs.mozilla.org/security/nss/legacy/key_log_format/index.html)
