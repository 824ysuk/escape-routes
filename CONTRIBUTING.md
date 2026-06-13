# Contributing

事例追加・既存事例の改善 PR を歓迎します。

## 新規事例の追加

1. `cases/NN-short-name/` directory を作成（番号は連番）
2. 以下 2 種類のファイルを置く:

**`cases/NN-short-name/README.md`**（事例の全体図）:

````markdown
# 事例 N: <タイトル>

## 何が起きるか
<直接ルートで詰まる症状の具体描写>

## 代替手段
| 手段 | 概要 | 再現性 / 用途 |
|---|---|---|
| [A. <手段名>](A-<short>.md) | ... | ... |
| [B. <手段名>](B-<short>.md) | ... | ... |

## 他手段を選ぶ条件
- B: <どんなとき B が合うか>
- C: <どんなとき C が合うか>

## 補足
<一行で全体観察>
````

**`cases/NN-short-name/X-<short>.md`**（各代替手段、A, B, ... ごとに 1 ファイル）:

````markdown
# 事例 N-X: <手段名>

## 前提 / install

install コマンド、必要なツール、認証情報、version 要件

## コード

```bash
<コピペで動く完全なコード>
```

## 期待出力

実行後に何が見えるか、成功 / 失敗の見分け方

## ハマりポイント

- OS / バージョン / 権限 / 制限事項
- セキュリティ警告（PII / token 漏洩等）が必要箇所に
````

3. トップ `README.md` の事例一覧表に行を追加（リンク先は `cases/NN-short-name/README.md`）
4. 共通パターンの該当事例番号を更新

## 既存事例の改善

- 誤字 / バージョン更新: 直接 PR
- 新しい代替手段の追加: 比較表に追記 + 個別ファイル（`X-<short>.md`）を新規作成
- ハマりポイントの追加: 該当代替手段ファイルの「ハマりポイント」項目に追記（過去事例も歓迎）

## 各代替手段の品質基準

- **前提 / install**: 必要なツールの install コマンドと version 要件が読み取れる
- **コード**: プレースホルダー（`YOUR_KEY` 等）はあって良いが、文法・引数・パイプは完全
- **期待出力**: 成功 / 失敗を目で見て判別できるレベル
- **ハマりポイント**: 実運用で典型的に遭遇する罠を網羅、セキュリティ警告（PII / token 漏洩等）が必要箇所に

## トーン

観察記録モード（事実 / 観察 / 参照元を並べる）。「ぜひ」「〜こそ」「重要です」等の説得語彙は使わない。

## Secret hygiene

事例には認証 token / Cookie / PII を扱うコード例が含まれる。PR 前に commit 漏洩を機械的に check する。

### .gitignore 共通エントリ

repo の `.gitignore` には以下を含める（既に含まれているものは確認）:

```gitignore
cookies.txt
*.har
state.json
token.json
device.json
long_token.json
sessions/
secrets.env
.env*
trace.zip
videos/
flows
*.mitm
*.pem
*.p12
*.key
```

### secret scanning

PR 前に [gitleaks](https://github.com/gitleaks/gitleaks) を走らせる:

```bash
brew install gitleaks   # macOS
gitleaks detect --source . --no-banner
```

### 万一 commit してしまったら

順序が重要。git history から削除する前に **対象 session / token を rotate** する。逆順では rotate 前に攻撃者が利用する。

1. 対象サービスの「他端末からログアウト」「API key 失効」「OAuth refresh_token revoke」を **最初に** 実行
2. 該当 session で行えた操作のログを確認（リポジトリ / 支払 / メール）
3. 必要なら所属組織の SOC / IT に通知
4. `git filter-repo` で history から削除 → force-push（rotate 後）

参考: [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) / [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
