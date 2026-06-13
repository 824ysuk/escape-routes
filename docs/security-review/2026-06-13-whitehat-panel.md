# 2026-06-13 ホワイトハッカーパネルレビュー

世界トップ級のセキュリティプログラマ 6 人格による escape-routes 全 case の adversarial review 結果。

## レビュー実施概要

- 実施日: 2026-06-13
- 対象: `README.md` / `CONTRIBUTING.md` / `cases/01-05/**/*.md` 全件
- 手法: Workflow による並列 adversarial review (6 persona 同時)
- 出力: 各 persona が finding を JSON schema で構造化、約 120 件
- 元 finding: `/private/tmp/claude-501/-Users-yusaku-Projects-escape-routes/.../tasks/wbp89aykk.output`

## 6 persona

| Persona | 専門領域 | 主な評価軸 |
|---|---|---|
| Moxie Marlinspike 風 | TLS / 通信観察 / 暗号 | TLS 1.3 / ECH / HTTP/3 QUIC / Cert Transparency / Android user CA / iOS Trust |
| Aaron Parecki + Daniel Fett 風 | OAuth 2.0 / OIDC / Cookie | RFC 9700 / RFC 8628 / refresh rotation / Bearer leakage / MV3 / PKCE / DPoP |
| Brendan Gregg + Greg KH 風 | Linux forensics / Profiling | ptrace_scope / perf_event_paranoid / systemd-coredump / strace overhead / eBPF |
| Wes McKinney + Hannes Mühleisen 風 | Data engineering / OOM | Node Streams backpressure / Polars / DuckDB / JVM MaxRAMPercentage / GNU parallel |
| ScrapFly / Apify 風 | Web scraping / Anti-bot | JA3/JA4 / curl_cffi / rebrowser / Cloudflare Turnstile / WAF / residential proxy |
| Bruce Schneier + Katie Moussouris 風 | Threat model / Legal / Ethics | Authorized Use / GDPR / 電気通信事業法 / 不正アクセス禁止法 / responsible disclosure |

## 横断テーマ (cross-cutting)

レビュー全体から抽出される、個別 finding を超えた構造的論点。

### T1. Authorized Use Only の宣言が repo 全体に存在しない (Critical)

`README.md` / `CONTRIBUTING.md` / 各 case README いずれにも「対象は利用者自身のアカウント / 自社が運用するサービス / 書面で許可された対象」という適用範囲宣言が無い。

扱う手法には (a) Cookie hijack 形式の curl、(b) 他人 SNS への scraping、(c) 他人モバイルアプリの mitmproxy 傍受、(d) 本番 PII の local 持ち出し が含まれる。AI agent が repo を読んで自動コード生成する状況も想定されており、dual-use 認識が repo 構造に組み込まれていないことが最大のリスク。

抵触しうる法域:
- 日本: 不正アクセス禁止法 3 条、電気通信事業法 4 条 (通信の秘密)、不正競争防止法 2 条 1 項 7 号・10 号
- 米国: CFAA (18 U.S.C. § 1030)、Wiretap Act (18 U.S.C. § 2511)
- EU: GDPR Art. 5・6・25
- UK: Computer Misuse Act 1990 §1

→ **本レビューで対応済** (R1: README に Scope / Disclosure section 追加、R2: 各 case README に scope 注記)。

### T2. PII / secret マスキングが deny-list 方式で構造的に脆弱 (Critical)

`cases/03-production-only-bug/A-trace-replay.md` (jq による Authorization / Cookie 除去) と `cases/05-invisible-traffic/E-debug-log.md` (Authorization / Cookie / X-API-Key の 3 key 固定) の双方で deny-list 方式を採用。

deny-list は新 auth header (DPoP RFC 9449 / HTTP Message Signatures RFC 9421 / x-amz-security-token / x-goog-api-key / x-ms-client-secret) を取りこぼす。GDPR Art. 5(1)(c) data minimization 原則からも allow-list が正解。

→ **本レビューで部分対応** (R5: 03/A を allow-list 例に書き換え)。05/E は findings 詳細参照。

### T3. 2024-2026 の anti-bot ランドスケープが反映されていない (High)

`cases/01-sns-dynamic-site/` 全般で:
- JA3 / JA4 / JA4+ TLS fingerprint への言及なし
- HTTP/2 / HTTP/3 fingerprint への言及なし
- curl-impersonate / curl_cffi の併記なし → 素 curl では Instagram 等で 403 多発
- playwright-stealth (Python) はメンテ低下 → rebrowser-playwright / patchright が現状の主軸
- WAF (Cloudflare Turnstile / DataDome / Akamai BMP / PerimeterX) の判定 cheat sheet なし
- residential proxy / mobile proxy の差なし
- EditThisCookie 拡張が 2024 年に売却・malware 化した経緯への注意喚起なし

→ **本レビューで部分対応** (R6: 01/B に EditThisCookie 撤回 + curl_cffi / JA3 注記)。残りは findings 詳細参照。

### T4. Linux kernel / container observability の現代的標準が反映されていない (High)

`cases/03-production-only-bug/` 全般で:
- `kernel.yama.ptrace_scope=0` への安易な変更を提示 → 1 に留めるべき
- `kernel.perf_event_paranoid` / `kptr_restrict` の前提が欠落
- `core_pattern` 上書きが systemd-coredump / apport の運用を破壊
- strace overhead の警告が弱い (実測 100x slowdown)
- eBPF (bpftrace / perf trace) が選択肢から完全欠落

→ **本レビューで部分対応** (R8: 03/B の ptrace_scope / perf_event_paranoid 修正)。03/D / 03/E は findings 詳細参照。

### T5. 2024 以降の OAuth / OIDC 規範が反映されていない (High)

`cases/02-authenticated-service/` 全般で:
- RFC 6749 単独引用 (RFC 9700 / OAuth 2.1 draft への移行に未言及)
- Device Flow polling back-off が RFC 8628 §3.5 と乖離 (interval=null fallback なし、上限なし)
- `verification_uri_complete` 未使用
- refresh_token rotation (RFC 9700 §2.2.2) 未言及
- Bearer / Cookie / TOTP secret を shell variable + 平文ファイルで扱い ps / history / artifact 漏洩を放置
- Playwright `tracing` / `record_video_dir` / screenshot に password / TOTP が記録される罠

→ **本レビューで部分対応** (R7: 02/B の RFC 8628 準拠化)。残りは findings 詳細参照。

---

## Critical 級 finding (本レビューで対応済)

### C1. README.md に Authorized Use Only 宣言なし

- Persona: Schneier
- 対応: README.md に `## 適用範囲 (Scope)` section と `## 脆弱性に気づいた場合 (Responsible Disclosure)` section を追加
- 法令参照: 不正アクセス禁止法 3 条、CFAA、Wiretap Act、GDPR Art. 6、Computer Misuse Act 1990 §1

### C2. 05/A mitmproxy の他人通信傍受境界なし

- Persona: Moxie, Schneier
- 対応: ファイル冒頭に「対象は自分の端末・自分のアカウント・自社アプリに限る」前提を追加
- 法令参照: 電気通信事業法 4 条、Wiretap Act、Computer Misuse Act 1990 §1
- 副次対応: Android `targetSdkVersion 24+` 条件、iOS の 2 段階手順、CA 削除手順、HTTP/3 (QUIC) 観察不可、Frida pinning bypass の倫理境界

### C3. 03/A trace-replay の PII deny-list が不完全

- Persona: Brendan, Schneier
- 対応: deny-list 例を残しつつ allow-list 例を主推奨に切替。退避先暗号化要件・保持 72h 上限・完了後 shred 削除を追記
- 法令参照: GDPR Art. 5(1)(c) / Art. 25 / Art. 32、改正個人情報保護法 20 条

---

## High 級 finding (本レビューで対応済)

### H1. 01/B EditThisCookie 推奨撤回 + curl_cffi 併記

- Persona: Aaron, Apify
- 対応: EditThisCookie を第一推奨から外し `Get cookies.txt LOCALLY` を昇格、curl-impersonate / curl_cffi の小節を追加、JA3/JA4 fingerprint の注意を追加

### H2. 02/B Device Flow の RFC 8628 準拠化

- Persona: Aaron
- 対応:
  - `interval=${INTERVAL:-5}` で省略時 5 秒 fallback
  - slow_down 時の上限 60 秒
  - expires_in 経過時の打切り処理
  - `verification_uri_complete` 優先表示
  - refresh_token rotation の書き戻し例
  - RFC 9700 / RFC 7009 (revocation) 参照

### H3. 03/B profiler の ptrace_scope=1 推奨

- Persona: Brendan
- 対応:
  - `ptrace_scope=0` の危険性を明示し `=1` をデフォルトに
  - `perf_event_paranoid` / `kptr_restrict` の前提を追加
  - py-spy の `--idle --native --subprocesses --rate 250` を推奨
  - K8s `kubectl debug --profile=sysadmin` を container 例として追加

### H4. 04/A-E の Streams / OOM 修正

- Persona: Wes
- 対応:
  - 04/A: `highWaterMark` を上げる説明を「pipeline() が自動処理」に書き換え。Polars LazyFrame / DuckDB / pandas chunksize の現代的比較
  - 04/E: GNU parallel の `--citation` プロンプト抑制 (`~/.parallel/will-cite`) と `--pipe-part --header :` での 1 ステップ化

---

## 本レビューで未対応 (要追って対応)

以下は session limit と修正 batch 量の都合で未着手。次回 PR で対応する候補。

### Aaron-Parecki (OAuth/Auth) 系

- **02/A-cookie-curl.md**: Bearer / CSRF を `curl --config` / `chmod 600 cookies.txt` / `set +o history` で安全に渡す例
- **02/A-cookie-curl.md**: Cookie の HttpOnly / Secure / SameSite 属性と取得可否
- **02/C-playwright.md**: `mask=[locator(...)]` で `screenshot()` / `tracing.start()` の機密フレーム除外
- **02/C-playwright.md**: TOTP の algorithm / digits / period / time skew (RFC 6238 §5)
- **02/D-mitmproxy.md**: CA の削除手順 (macOS Keychain / Linux update-ca-certificates / Windows MMC)、`~/.mitmproxy/` の `chmod 700`、CSRF/state/PKCE は one-shot で再現不可
- **02/E-extension.md**: Manifest V3 の `webRequest` observe-only、`declarativeNetRequest` 移行、sideload 警告
- **02 README.md**: 比較表に「正当性」列追加 (Device Flow = 公式許諾)

### Moxie (TLS) 系

- **05/A-mitmproxy.md**: TLS 1.3 復号機構の説明、CT (RFC 9162) 文脈での CA 注入副作用、mitmweb/mitmdump/mitmproxy 使い分け
- **05/B-charles-proxyman.md**: macOS proxy off 確認 (`scutil --proxy` / `networksetup -getwebproxy`)
- **05/C-wireshark.md**: SSLKEYLOGFILE の TLS 1.3 形式 (5 種 SECRET)、capture filter (BPF) vs display filter の区別、macOS `-i any` 制約 (BSD 設計起因 / pktap 代替)
- **05/D-mobile-devtools.md**: Android Studio Hedgehog 以降の Network Inspector への移行、iOS Web Inspector は WKWebView のみ対象 (native URLSession 対象外)
- **05/E-debug-log.md**: ヘッダマスクを allow-list 化、JWT header 部の選択的開示
- **05 README.md**: ECH (RFC 9460) で SNI が見えなくなる条件

### Brendan (Linux forensics) 系

- **03/D-core-dump.md**: systemd-coredump / apport の hijack 警告、container 用 dedicated セクション (`--ulimit core=-1`、K8s DaemonSet)、`limits.conf` は systemd 配下で効かない事実を冒頭に
- **03/D-core-dump.md**: Go の `GOTRACEBACK=crash` で release binary のまま core 取得、gdb libpython 拡張 (`py-bt` / `py-locals`)
- **03/E-tcpdump-strace.md**: strace overhead 警告強化、`perf trace` / `bpftrace` を第一選択に、tcpdump 4.99+ の pcap-ng 既定、`-i any` の Linux Cooked Capture v2 制約
- **03/C-canary.md**: Argo Rollouts AnalysisTemplate の具体 YAML、error budget / blast radius 上限
- **03 README.md**: eBPF / `bpftrace` / `perf record + flamegraph` を F として追加

### Wes (Data engineering) 系

- **04/A-stream.md**: Node v18 → v22 LTS 更新、JSONL を本文に昇格、Parquet / Arrow IPC への言及
- **04/B-memory-limit.md**: JVM `-XX:MaxRAMPercentage`、cgroup v2 unified hierarchy、`jcmd GC.heap_dump` (jmap deprecated)
- **04/C-sqlite-duckdb.md**: `read_csv_auto` → `read_csv` (DuckDB v0.10+)、SQLite PRAGMA (WAL / synchronous / cache_size)、DuckDB の httpfs / Parquet 直接クエリ、OLAP vs OLTP positioning
- **04/D-other-language.md**: Go GC vs Rust borrow checker の trade-off、Polars/DuckDB が「Rust を書く」前段の選択肢
- **04/E-parallel-split.md**: `mlr` / `xsv` / `csvkit` の選択肢比較、Dask / Ray data の TB 級代替
- **04 README.md**: データ形状ベースの分岐フロー (集計→DuckDB / 変換→stream / TB+→Dask)

### Apify (Web scraping) 系

- **01 README.md**: 「検知の 6 層」(IP / TLS / H2 / H3 / JS fingerprint / behavioral) 追加、「法的 3 層」(技術 / 契約 / データ保護) 追加、比較表に「成功率」「コスト」列
- **01/A-browser-console.md**: Instagram CDN URL の `oe=` (hex)、IP / referrer binding の訂正
- **01/B-cookie-curl.md**: `X-ASBD-ID` / `X-IG-WWW-Claim` 等の必須ヘッダ、最小必須 cookie 表 (`mid` / `ig_did` / `ds_user_id`)
- **01/C-specialized-oss.md**: shadowban のシグナルと回復手順 (`429` / `checkpoint_required` / 24-72h 停止)
- **01/D-official-api.md**: Graph API バージョンの 2 年 deprecate サイクル、`ig_refresh_token`、`/me/permissions` revoke、Business Discovery / hashtag search の制約マトリクス
- **01/E-headless-stealth.md**: rebrowser-playwright / patchright / camoufox / nodriver の比較、bot.sannysoft.com 等の検証 URL、stealth が消すべき 8 項目、`wait_until='networkidle'` を要素ベース待機に
- **01/F-commercial-api.md**: ScrapingBee / ScrapFly / Bright Data / Apify / Zyte / Oxylabs の比較表、proxy 種別 (datacenter / ISP / residential / mobile)、Librahack 事件 (req/sec 自己抑制)、CAPTCHA solver の対応マトリクス

### Schneier (Threat model / Legal) 系

- **02/E-extension.md**: sideload (デベロッパーモード) 警告、Unlisted / Enterprise Policy 配布の選択肢
- **CONTRIBUTING.md**: 各代替手段ファイルの metadata block (対象範囲 / 第三者データ / PII / 適法境界 / disclosure 経路) 必須化

---

## 次のアクション提案

1. **本 PR をマージ後**、未対応 finding を機能別に 5-6 個の Issue に分割 (Apify 系 / Brendan 系 / Aaron 系 / Moxie 系 / Wes 系) して segment 化
2. 各 Issue に対応 PR を立てる際は本 review の元 finding (`personaResults[].findings`) を引用元として参照
3. `improve-claude-md` / `behavioral-audit` skill の review 対象に本ファイルを追加 (定期再評価)

## 元 finding 全文

Workflow 出力の JSON は worktree 外の `/private/tmp/claude-501/-Users-yusaku-Projects-escape-routes/2de3ed07-10af-4760-a4ae-4b27a460cc26/tasks/wbp89aykk.output` に残るが、tmpfs のため session 終了で揮発する可能性あり。重要な引用箇所は本ファイル「未対応」セクションに転記済み。
