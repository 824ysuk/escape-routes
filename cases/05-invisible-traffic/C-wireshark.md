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
export SSLKEYLOGFILE=$HOME/.ssl-key.log
open -a "Google Chrome"      # macOS
# Wireshark → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename に $SSLKEYLOGFILE を設定
```

## コード

```bash
# GUI: 特定インターフェース + capture filter で起動
sudo wireshark -i en0 -k -f 'host api.example.com and tcp port 443'

# CLI 版 tshark: TLS handshake から接続先 SNI を列挙
sudo tshark -i any -f 'tcp port 443' -Y 'tls.handshake.type == 1' -T fields -e tls.handshake.extensions_server_name

# 実出力例:
# api.example.com
# cdn.example.com

# 失敗パターン抽出（RST = 接続リセット）
sudo tshark -i any -Y 'tcp.flags.reset == 1' -T fields -e ip.src -e ip.dst -e tcp.dstport

# pcap として保存 → 後で GUI で読む
sudo tshark -i any -f 'host api.example.com' -w /tmp/cap.pcap
```

## 期待出力

- GUI: パケット一覧（時刻 / src / dst / protocol / info）+ 詳細ペイン
- tshark: 1 行 1 record の TAB 区切り出力（SNI / IP / port など）
- `SSLKEYLOGFILE` 経由で Chrome の TLS を復号した場合: Wireshark の「Decrypted TLS」ペインで HTTPS body が見える

## ハマりポイント

- HTTPS の body は鍵がなければ復号不可能 → アプリのソースが手元になければ `SSLKEYLOGFILE` 経由しか道なし
- 大量パケットの pcap は GUI フィルタが遅い。CLI で先に絞ってから GUI で開く
- macOS の SIP 制約で `any` インターフェースが使えないことがある → `-i en0` 等の個別指定
- `SSLKEYLOGFILE` は機密情報 → 公開リポジトリに含めない、使用後に削除
