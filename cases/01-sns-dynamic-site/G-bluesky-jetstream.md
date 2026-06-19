# 事例 1-G: Bluesky Jetstream（リアルタイム購読）

> **適用条件**: Bluesky（AT Protocol）の公開 firehose に限る。event に署名が付かないため、再検証可能アーカイブ用途には official CBOR firehose を使う（下記「ハマりポイント」参照）。詳細はトップ [README.md](../../README.md#適用範囲-scope)。

## 前提 / install

- 公開 endpoint（US 4 リージョン）:
  - `jetstream1.us-east.bsky.network`
  - `jetstream2.us-east.bsky.network`
  - `jetstream1.us-west.bsky.network`
  - `jetstream2.us-west.bsky.network`
- WebSocket path: `/subscribe`
- 認証なし（public read-only stream）
- クライアント: [`websocat`](https://github.com/vi/websocat)（最速）/ Python `websockets` / Node `ws` のいずれか

```bash
# websocat
brew install websocat

# Python websockets
pip install websockets
```

## コード

### websocat 1 行（post 全件購読）

```bash
websocat 'wss://jetstream2.us-east.bsky.network/subscribe?wantedCollections=app.bsky.feed.post'
```

### filter 複数指定（post + like + 特定 DID）

```bash
websocat 'wss://jetstream2.us-east.bsky.network/subscribe?wantedCollections=app.bsky.feed.post&wantedCollections=app.bsky.feed.like&wantedDids=did:plc:eygmaihciaxprqvxpfvl6flk'
```

### Python（`websockets` で commit event だけ抜き出す）

```python
import asyncio, json, websockets

URL = ("wss://jetstream2.us-east.bsky.network/subscribe"
       "?wantedCollections=app.bsky.feed.post")

async def main():
    async with websockets.connect(URL, max_size=None) as ws:
        async for msg in ws:
            evt = json.loads(msg)
            if evt.get("kind") == "commit":
                c = evt["commit"]
                print(c["operation"], c["collection"], c["rkey"])

asyncio.run(main())
```

## 期待出力

`kind: commit`（post 作成イベント、`record` は省略形）:

```json
{
  "did": "did:plc:eygmaihciaxprqvxpfvl6flk",
  "time_us": 1725911162329308,
  "kind": "commit",
  "commit": {
    "rev": "3l3qo2vutsw2b",
    "operation": "create",
    "collection": "app.bsky.feed.like",
    "rkey": "3l3qo2vuowo2b",
    "record": { "$type": "app.bsky.feed.like", "createdAt": "2024-09-09T19:46:02.102Z",
                "subject": { "cid": "bafyrei...", "uri": "at://did:plc:.../app.bsky.feed.post/3l3pte3p2e325" } },
    "cid": "bafyrei..."
  }
}
```

`kind: identity`（handle 変更通知）:

```json
{
  "did": "did:plc:ufbl4k27gp6kzas5glhz7fim",
  "time_us": 1725516665234703,
  "kind": "identity",
  "identity": { "did": "did:plc:ufbl4k27gp6kzas5glhz7fim", "handle": "yohenrique.bsky.social",
                "seq": 1409752997, "time": "2024-09-05T06:11:04.870Z" }
}
```

`kind: account`（アカウント active/inactive 切り替え）:

```json
{
  "did": "did:plc:ufbl4k27gp6kzas5glhz7fim",
  "time_us": 1725516665333808,
  "kind": "account",
  "account": { "active": true, "did": "did:plc:ufbl4k27gp6kzas5glhz7fim",
               "seq": 1409753013, "time": "2024-09-05T06:11:04.870Z" }
}
```

## ハマりポイント

- **filter の上限とサーバ側適用**: `wantedDids` は最大 10,000、`wantedCollections` は最大 100。上限超過 query の挙動（明示エラー / 黙殺切り捨て）はクライアント側でテストして確認する
- **prefix は NSID 境界でしか効かない**: `app.bsky.graph.*` や `app.bsky.*` は OK だが、`app.bsky.graph.fo*` のような途中 prefix は unsupported。filter 設計時に NSID の階層を意識する
- **event は unsigned**: Jetstream の JSON は relay 側で CBOR firehose から派生変換されたもので署名が落ちている。後段で「本人が確かに投稿した」と再検証する用途（archival / 法的証跡）には official CBOR firehose に切り替える
- **`compress=true` は zstd dictionary 同梱が必要**: `compress=true` または `Socket-Encoding: zstd` header で zstd 圧縮になるが、Jetstream リポジトリ同梱の dictionary（`pkg/models/zstd_dictionary`）を読み込まないと frame をデコードできない。標準 zstd ライブラリの素の decoder を当てると失敗する
- **`cursor` は unix microseconds**: 再接続時は最後に処理した event の `time_us` から少し負方向にバッファした値を渡す（gapless replay）。absent / 未来時刻の cursor は live-tail にフォールバックする
