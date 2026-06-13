# 事例 3-A: trace ID から request payload を抜いて local replay

> **対象範囲**: 本番 PII を local に持ち出す行為は GDPR Art. 5(1)(c) data minimization / Art. 25 data protection by design / Art. 32 security of processing、改正個人情報保護法 20 条 (安全管理措置) の対象。実施前に DPIA (data protection impact assessment) または社内 privacy review を経て、退避先暗号化・保持期間上限・完了後削除を決めてから手順に進む。退避先は **FileVault (macOS) / dm-crypt (Linux) / BitLocker (Windows) で暗号化されたディスク**、保持は **72 時間上限**、完了後 `shred -u` で削除する。

## 前提 / install

- APM ツールが request body を保存していることを確認:
    - Datadog APM: `apm.span.body_capture` を datadog-agent.yaml で有効化
    - New Relic: `attributes.include = request.headers.*` / `request.body` を newrelic.yml で
    - Sentry: tracing は payload を保存しない（breadcrumb 経由）
- 取り込み段階のマスクを優先: Datadog [`obfuscation_config`](https://docs.datadoghq.com/data_security/agent/) / New Relic `attributes.exclude` を設定し、unsanitized PII が APM index に流れない状態にする
- API key 取得 → 環境変数 `DD_API_KEY` / `DD_APP_KEY` / `TARGET_TRACE_ID` を export
- `jq` install
- local server を本番と同 image で起動: `docker pull registry.example.com/api:v1.2.3 && docker run -p 3000:3000 ...`

## コード

```bash
# 0. 作業ディレクトリを 700 / 暗号化済みボリューム配下に
umask 077
WORK=$(mktemp -d -t trace-replay)
cd "$WORK"

# 1. APM から該当 trace を抽出
curl -X POST 'https://api.datadoghq.com/api/v2/logs/events/search' \
  -H "DD-API-KEY: $DD_API_KEY" \
  -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  -H 'Content-Type: application/json' \
  -d "{\"filter\":{\"query\":\"trace_id:${TARGET_TRACE_ID} status:error\",\"from\":\"now-1h\",\"to\":\"now\"}}" \
  -o trace.json

# 2. request body を抽出（field path は APM の log structure 依存）
jq '.data[0].attributes.attributes.http.request_body' trace.json > payload.json

# 3a. PII / secret サニタイズ（allow-list 方式を default に）
#     再現に必要な field のみ通す。未知の field は混入させない。
jq '{
  method: .method,
  path: .path,
  query: { page: .query.page, limit: .query.limit },
  body: {
    orderId: .body.orderId,
    sku: .body.sku,
    quantity: .body.quantity,
    promoCode: .body.promoCode
  }
}' payload.json > payload-allowlist.json

# 3b. allow-list で抽出できない場合（高 cardinality な動的 field 等）の補助:
#     deny-list で広めに削る + 残存 secret の regex 検出を 2 段で
jq '
  del(.headers.Authorization) | del(.headers.authorization) |
  del(.headers.Cookie) | del(.headers.cookie) |
  del(.headers["X-API-Key"]) | del(.headers["X-CSRF-Token"]) |
  del(.headers["X-Amz-Security-Token"]) | del(.headers["X-Goog-Api-Key"]) |
  del(.headers["X-Ms-Client-Secret"]) | del(.headers["Proxy-Authorization"]) |
  del(.headers["DPoP"]) | del(.headers["Signature"]) |
  del(.user.email) | del(.user.phone) | del(.user.address) |
  del(.user.ip) | del(.user.device_id) | del(.user.session_id) |
  walk(if type == "object" and has("password")      then .password      = "***" else . end) |
  walk(if type == "object" and has("apiKey")        then .apiKey        = "***" else . end) |
  walk(if type == "object" and has("refresh_token") then .refresh_token = "***" else . end) |
  walk(if type == "object" and has("private_key")   then .private_key   = "***" else . end)
' payload.json > payload-denylist.json

# 3c. 残存 secret の検出（JWT / base64 / API key 風文字列）
# JWT 検出
grep -oE '[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{20,}' payload-allowlist.json && \
  echo "WARN: JWT-like string detected, review allow-list" >&2

# 4. local server に投げて再現
curl -X POST http://localhost:3000/api/v1/orders \
  -H 'Content-Type: application/json' \
  -d @payload-allowlist.json \
  -w 'http_code=%{http_code} time=%{time_total}\n' \
  -o local-response.json

# 5. error stack 照合
PROD_STACK=$(jq -r '.data[0].attributes.attributes.error.stack' trace.json)
LOCAL_STACK=$(jq -r '.error.stack' local-response.json 2>/dev/null || echo '')
diff <(echo "$PROD_STACK") <(echo "$LOCAL_STACK") && echo '同一 stack で再現'

# 6. 完了後 (≦72h で必ず)
shred -u "$WORK"/*.json
rmdir "$WORK"
```

## 期待出力

- `trace.json` に該当 trace の log entries
- `payload-allowlist.json` に再現に必要な field のみ抽出されたデータ (PII は含まない設計)
- `http_code=500` + `local-response.json` に error stack
- `diff` の出力で同一 stack（再現成功）/ 差分（再現失敗）が判別

## ハマりポイント

### マスク方式

- **allow-list を default にする**。deny-list は新 auth header (DPoP [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449) / HTTP Message Signatures [RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421) / `x-amz-security-token` / `x-goog-api-key`) を取りこぼす
- 「個人情報」は氏名以外の組み合わせ (IP + UA + timestamp) でも該当しうる (改正個人情報保護法 2 条 1 項)。allow-list で明示的に許可した field のみ通す
- より厳密にやるなら [Microsoft Presidio](https://microsoft.github.io/presidio/) / [scrubadub](https://github.com/datascopeanalytics/scrubadub) で 2 段サニタイズ

### APM 設定の罠

- APM によってはコスト削減で body 未保存。事前に APM 設定を確認
- body capture を一時的に有効化する判断は decision log 対象 (`docs/decisions/`)。GDPR Art. 17 (right to erasure) が来たとき APM index の purge も対象に含める

### local 環境の限界

- local の DB 状態が production と違うと再現できない。本番 DB の sanitized dump を別途 restore する場合、`mysqldump` / `pg_dump` の段階で PII を masked column に置換する (例: [Martin Fowler "Synthetic Data"](https://martinfowler.com/articles/data-mesh-principles.html) 参照)
- 個人 data を含む dump は **絶対に Slack / Notion / メール / 共有ドライブに置かない**。退避先は暗号化ディスクのみ

### 退避先 / 保持期間

- 退避先: FileVault (macOS) / dm-crypt (Linux) / BitLocker (Windows) で暗号化されたディスク
- 保持期間: 72 時間上限、完了後 `shred -u`
- audit log: 誰が・いつ・何の trace を抽出したかを社内チケットに記録
