# escape-routes

直接ルート（curl / fetch / アプリを動かす）が詰まったときの代替手段カタログ。
1 事例 = 1 ファイルで、各代替手段に「前提 / install → コード → 期待出力 → ハマりポイント」を揃える。

## なぜこの repo か

- 領域内に閉じた裏技集（[awesome-blackmagic](https://github.com/tnfe/awesome-blackmagic) / [Awesome-Hacking](https://github.com/Hack-with-Github/Awesome-Hacking)）と、問題解決の理論書（[How to Solve It (Polya)](https://en.wikipedia.org/wiki/How_to_Solve_It) / [Lateral Thinking (de Bono)](https://en.wikipedia.org/wiki/Lateral_thinking)）は存在するが、その間（**領域横断 × 具体事例**）は手薄。
- 「動いた = 最適」ではない。再現性・学習コスト・コストで使い分けられるように、**採用しなかった手段も併記** する。
- 個人で育てる前提。1 事例 = 1 PR で追加していく。

## 事例一覧

| # | 事例 | 詰まる原因 | 代替手段数 |
|---|---|---|---|
| 1 | [SNS / 動的サイトからデータが取れない](cases/01-sns-dynamic-site.md) | ボット検知・JS 動的レンダリング | 6 |
| 2 | [認証必須サービスから定期取得したい](cases/02-authenticated-service.md) | OAuth / 2FA / Cookie 同期 | 5 |
| 3 | [本番でしか起きないバグを調査したい](cases/03-production-only-bug.md) | local で再現できない | 5 |
| 4 | [巨大データで OOM が出る](cases/04-big-data-oom.md) | メモリに収まらない | 5 |
| 5 | [ブラウザに見えない通信を観察したい](cases/05-invisible-traffic.md) | モバイル・他アプリ・暗号化通信 | 5 |

## 共通パターン

5 事例から抽出される、代替手段を考えるときの発想軸。

| パターン | 一言 | 該当事例 |
|---|---|---|
| レイヤーを動かす | HTTP → DOM、API → DB、アプリログ → syscall。抽象を上下させる | 1, 3, 5 |
| ブラウザ session を使う | CLI から再現困難な認証を、ブラウザ経由で済ませる | 1, 2 |
| 状態を運んで再現 | local 環境を整える代わりに、本番の状態を持ってくる | 3 |
| 問題を分割する | 全体を一度に解かない。stream・チャンク・分業 | 4 |
| 間接ルートに切り替える | 直接アクセスを諦め、proxy・観察ツール・API 経由にする | 2, 5 |

## 各事例の最小スキーマ

新規事例を追加するときは以下の構造で書く:

```
タイトル: <事例タイトル>
詰まる原因: <カテゴリ>
直接ルート: <試みた直接アプローチ>
代替手段:
  - 名前 / 前提・install / 最小コード / 期待出力 / ハマりポイント / リンク
採用手段: <どれか>
採用理由: <再現性 or 学習コスト or コスト>
補足: <一行>
```

## Contributing

PR で事例追加・既存事例の改善を歓迎します。詳細は [CONTRIBUTING.md](CONTRIBUTING.md)。

## 参考リソース

### 領域内の裏技・bypass 集
- [tnfe/awesome-blackmagic](https://github.com/tnfe/awesome-blackmagic) — Web 開発の "黒魔法" コレクション（中国語）
- [Hack-with-Github/Awesome-Hacking](https://github.com/Hack-with-Github/Awesome-Hacking) — セキュリティ系の bypass / 攻撃手法集
- [enaqx/awesome-pentest](https://github.com/enaqx/awesome-pentest) — penetration testing リソース集

### 著名 engineer の実例集
- [Julia Evans](https://jvns.ca/) — debugging / systems / networking
- [Brendan Gregg](https://www.brendangregg.com/) — performance / observability。flamegraph 発案者
- [Dan Luu](https://danluu.com/) — システム・実例の深掘り
- [Sean Goedecke](https://www.seangoedecke.com/) — pragmatic engineer のブログ

### 問題解決の古典
- [How to Solve It (Wikipedia)](https://en.wikipedia.org/wiki/How_to_Solve_It) — George Pólya、1945
- [Lateral Thinking (Wikipedia)](https://en.wikipedia.org/wiki/Lateral_thinking) — Edward de Bono、1967
- [The Pragmatic Programmer](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) — David Thomas & Andrew Hunt

## License

MIT
