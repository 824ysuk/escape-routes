# 事例 3-A: trace ID から request payload を抜いて local replay

## 前提 / install

- APM ツールが request body を保存していることを確認:
    - Datadog APM: `apm.span.body_capture` を datadog-agent.yaml で有効化
    - New Relic: `attributes.include = request.headers.*` / `request.body` を newrelic.yml で
    - Sentry: tracing は payload を保存しない（breadcrumb 経由）
- API key 取得 → 環境変数 `DD_API_KEY` / `DD_APP_KEY` / `TARGET_TRACE_ID` を export
- `jq` install
- local server を本番と同 image で起動: `docker pull registry.example.com/api:v1.2.3 && docker run -p 3000:3000 ...`

## コード

```bash
# 1. APM から該当 trace を抽出
curl -X POST 'https://api.datadoghq.com/api/v2/logs/events/search' -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" -H 'Content-Type: application/json' -d "{\"filter\":{\"query\":\"trace_id:${TARGET_TRACE_ID} status:error\",\"from\":\"now-1h\",\"to\":\"now\"}}" -o trace.json

# 2. request body を抽出（field path は APM の log structure 依存）
jq '.data[0].attributes.attributes.http.request_body' trace.json > payload.json

# 3. PII / secret マスク checklist
jq 'del(.headers.Authorization) | del(.headers.Cookie) | del(.headers["X-API-Key"]) | del(.headers["X-CSRF-Token"]) | del(.user.email) | del(.user.phone) | del(.user.password) | del(.payment.card_number) | del(.payment.cvv) | walk(if type == "object" and has("password") then .password = "***" else . end) | walk(if type == "object" and has("apiKey") then .apiKey = "***" else . end) | walk(if type == "object" and has("refresh_token") then .refresh_token = "***" else . end)' payload.json > payload-sanitized.json

# 4. local server に投げて再現
curl -X POST http://localhost:3000/api/v1/orders -H 'Content-Type: application/json' -d @payload-sanitized.json -w 'http_code=%{http_code} time=%{time_total}\n' -o local-response.json

# 5. error stack 照合
PROD_STACK=$(jq -r '.data[0].attributes.attributes.error.stack' trace.json)
LOCAL_STACK=$(jq -r '.error.stack' local-response.json 2>/dev/null || echo '')
diff <(echo "$PROD_STACK") <(echo "$LOCAL_STACK") && echo '同一 stack で再現'
```

## 期待出力

- `trace.json` に該当 trace の log entries
- `payload-sanitized.json` に PII を除いた request body
- `http_code=500` + `local-response.json` に error stack
- `diff` の出力で同一 stack（再現成功）/ 差分（再現失敗）が判別

## ハマりポイント

- APM によってはコスト削減で body 未保存。事前に APM 設定を確認
- PII / secret は **header（Authorization / Cookie / X-API-Key / X-CSRF-Token）+ body（password / apiKey / refresh_token / token / secret）+ query string** を網羅的にマスク
- local の DB 状態が production と違うと再現できない → 本番 DB の sanitized dump を別途 restore
