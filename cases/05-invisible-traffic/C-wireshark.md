# 事例 5-C: Wireshark / tshark

## 前提 / install

```bash
brew install --cask wireshark   # macOS
sudo apt install wireshark tshark   # Linux

# パケットキャプチャ権限（sudo 不要にする）
sudo dseditgroup -o edit -a $USER -t user access_bpf   # macOS
sudo usermod -aG wireshark $USER                       # Linux（再ログイン要）
```

- 自分のブラウザの TLS 通信を Wireshark で復号したい場合は環境変数 `SSLKEYLOGFILE` を設定 → Chrome / Firefox を起動 → Wireshark の Preferences → Protocols → TLS で同じファイルを指定:

```bash
# このセッション内のみ。.zshrc / .bash_profile への永続化は禁止。
# 永続化すると以降の全ブラウザ通信の TLS 鍵が同ファイルに append され続け、
# 鍵 + キャプチャがセットで残ると過去通信が事後復号可能になる。
export SSLKEYLOGFILE=$HOME/.ssl-key.log
chmod 600 "$SSLKEYLOGFILE" 2>/dev/null || :
open -a "Google Chrome"      # macOS
# Wireshark → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename に $SSLKEYLOGFILE を設定
```

## コード

```bash
# GUI: 特定インターフェース + capture filter で起動
# HTTP/3 (QUIC) も含めるなら UDP 443 を追加 (HTTP/3 移行済 API ではこちらが主経路)
sudo wireshark -i en0 -k -f 'host api.example.com and ((tcp or udp) port 443)'

# CLI 版 tshark: TLS handshake から接続先 SNI を列挙
sudo tshark -i any -f 'tcp port 443' -Y 'tls.handshake.type == 1' -T fields -e tls.handshake.extensions_server_name

# HTTP/3 (QUIC) の SNI を取る (TLS handshake は UDP 上で走るため tls.* フィルタには出ない)
sudo tshark -i any -Y 'quic.tls.handshake.type == 1' -T fields -e quic.tls.handshake.extensions_server_name

# 実出力例:
# api.example.com
# cdn.example.com

# 失敗パターン抽出（RST = 接続リセット）
sudo tshark -i any -Y 'tcp.flags.reset == 1' -T fields -e ip.src -e ip.dst -e tcp.dstport

# pcap として保存 → 後で GUI で読む
install -d -m 700 "$HOME/.captures/$(date +%Y%m%d)"
sudo tshark -i any -f 'host api.example.com' -w "$HOME/.captures/$(date +%Y%m%d)/cap.pcap"
```

## 期待出力

- GUI: パケット一覧（時刻 / src / dst / protocol / info）+ 詳細ペイン
- tshark: 1 行 1 record の TAB 区切り出力（SNI / IP / port など）
- `SSLKEYLOGFILE` 経由で Chrome の TLS を復号した場合: Wireshark の「Decrypted TLS」ペインで HTTPS body が見える

## ハマりポイント

- HTTPS の body は鍵がなければ復号不可能 → アプリのソースが手元になければ `SSLKEYLOGFILE` 経由しか道なし
- 大量パケットの pcap は GUI フィルタが遅い。CLI で先に絞ってから GUI で開く
- macOS の SIP 制約で `any` インターフェースが使えないことがある → `-i en0` 等の個別指定
- **TLS 1.3 ECH (Encrypted Client Hello)** 有効な接続（Chrome の DNS over HTTPS + ECH が効くサイト、Cloudflare backend 等）では outer ClientHello の SNI が空またはダミー（`cloudflare-ech.com` 等）になり、real SNI は暗号化される。`tls.handshake.extensions.encrypted_client_hello` フィルタで ECH 有無を判定（[draft-ietf-tls-esni](https://datatracker.ietf.org/doc/draft-ietf-tls-esni/) / [Cloudflare announcement](https://blog.cloudflare.com/announcing-encrypted-client-hello/)）
- **HTTP/3 (QUIC)** は TLS handshake が UDP 443 上で走り、`tcp port 443` filter / `tls.handshake.*` フィルタには出ない。capture filter に `(tcp or udp) port 443`、display filter に `quic.tls.handshake.*` を使う（[Wireshark quic fields](https://www.wireshark.org/docs/dfref/q/quic.html) / [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000)）
- **pcap / SSLKEYLOGFILE の取り扱い**: pcap には DNS query / 認証 Cookie（TLS 復号後）/ Authorization / PII / TCP payload が含まれる。`SSLKEYLOGFILE` はパスワード相当の機密で、鍵 + キャプチャがセットで残ると過去通信が事後復号可能。書き出し先は `~/.captures/YYYYMMDD/`（700）に固定し、解析後 `shred -u` または `rm -P`（macOS）で破棄。Slack / Issue に貼るときは `editcap -F pcap -s 64 in.pcap out.pcap` で payload を切り詰める（[editcap docs](https://www.wireshark.org/docs/man-pages/editcap.html) / [Wireshark TLS docs](https://wiki.wireshark.org/TLS#using-the-pre-master-secret)）
- `SSLKEYLOGFILE` 後始末: (1) ファイル削除 → (2) `unset SSLKEYLOGFILE` → (3) シェル再起動で残留がないことを確認
