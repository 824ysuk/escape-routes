# 事例 3-A: trace ID から request payload を抜いて local replay

## 前提 / install

- APM ツールが request body を保存していることを確認（各 APM の収集経路）:
    - Datadog APM: Tracer SDK の RequestBody hook または AAP (Application & API Protection) / RUM 経由（agent yaml の `apm.span.body_capture` 単独で body は収集されない、[公式 docs](https://docs.datadoghq.com/tracing/configure_data_security/) 参照）
    - New Relic: `attributes.include = request.headers.*` で header を、body は `record_custom_event` / `add_custom_parameter` で手動付与（[custom attributes docs](https://docs.newrelic.com/docs/apm/agents/manage-apm-agents/agent-data/collect-custom-attributes/)）
    - Sentry: `send_default_pii=True` + `max_request_body_size='medium'` で収集（[Sentry options](https://docs.sentry.io/platforms/python/configuration/options/#max-request-body-size)）
- API key 取得 → 環境変数 `DD_API_KEY` / `DD_APP_KEY` / `TARGET_TRACE_ID` を export
- `jq` install、`gitleaks` または `trufflehog` を入れておく（派生 file の secret 二次検査用）
- local server を本番と同 image で起動: `docker pull registry.example.com/api:v1.2.3 && docker run -p 3000:3000 ...`

## コード

```bash
# 1. APM から該当 trace を抽出 (Spans Events Search API)
# field path はテナント設定依存。log search を使う場合は /api/v2/logs/events/search に差し替える
curl -X POST 'https://api.datadoghq.com/api/v2/spans/events/search' \
  -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  -H 'Content-Type: application/json' \
  -d "{\"data\":{\"attributes\":{\"filter\":{\"query\":\"@trace_id:${TARGET_TRACE_ID} status:error\",\"from\":\"now-1h\",\"to\":\"now\"},\"page\":{\"limit\":50},\"sort\":\"timestamp\"}}}" \
  -o trace.json

# 2. request body を抽出（field path は APM の log structure 依存。テナント設定により
#    .data[].attributes.custom.http.request.body などにマップされていることもある）
jq '.data[0].attributes.attributes.http.request_body' trace.json > payload.json

# 3. PII / secret マスク (allowlist 方式)
#    denylist は未列挙キーが必ず漏れるため、「明示的に許可する key だけを残し、
#    それ以外の primitive 値はキー名のパターンに応じて *** にする」allowlist 戦略にする。
#    masking 対象キーは正規表現で網羅。明示 allowlist で許可しない object は丸ごと捨てる。
ALLOWED_TOP_KEYS='["order_id","item_id","quantity","currency","locale","client_version"]'
SENSITIVE_KEY_RE='(?i)(pass|pwd|secret|token|apikey|api_key|authorization|cookie|csrf|email|phone|mobile|ssn|dob|birth|card|pan|cvv|cvc|address|zipcode|name|account)'
jq --argjson allow "$ALLOWED_TOP_KEYS" --arg re "$SENSITIVE_KEY_RE" '
  def mask:
    walk(
      if type == "object" then
        with_entries(
          if (.key | test($re)) then .value = "***"
          else .
          end
        )
      else .
      end
    );
  # トップ階層は allowlist で絞る (未知の key は丸ごと捨てる)
  to_entries | map(select(.key as $k | $allow | index($k))) | from_entries
  | mask
' payload.json > payload-sanitized.json

# 派生 file を secret scanner で 2 重チェック
gitleaks detect --no-git --source payload-sanitized.json --report-format=json --report-path=gitleaks-report.json || \
  { echo 'secret 検出。masking が漏れている'; exit 1; }

# 4. local server に投げて再現
curl -X POST http://localhost:3000/api/v1/orders -H 'Content-Type: application/json' -d @payload-sanitized.json -w 'http_code=%{http_code} time=%{time_total}\n' -o local-response.json

# 5. error stack 照合
PROD_STACK=$(jq -r '.data[0].attributes.attributes.error.stack' trace.json)
LOCAL_STACK=$(jq -r '.error.stack' local-response.json 2>/dev/null || echo '')
diff <(echo "$PROD_STACK") <(echo "$LOCAL_STACK") && echo '同一 stack で再現'

# 6. 派生 file の破棄 (原本 + sanitized + report)
shred -u trace.json payload.json gitleaks-report.json 2>/dev/null || rm -P trace.json payload.json gitleaks-report.json
```

## 期待出力

- `trace.json` に該当 trace の log entries
- `payload-sanitized.json` に PII を除いた request body
- `http_code=500` + `local-response.json` に error stack
- `diff` の出力で同一 stack（再現成功）/ 差分（再現失敗）が判別

## ハマりポイント

- APM によってはコスト削減で body 未保存。事前に APM 設定を確認
- PII / secret マスクは **denylist 禁止**。`del(.headers.Authorization)` のような既知キー列挙は `email_addr` / `phoneNumber` / `card.pan` / `oldPassword` / `pwd` 等の未列挙キーが必ず漏れる。allowlist (許可する key だけ残す) + 正規表現マスク + `gitleaks` / `trufflehog` での二重検出が最低ライン（[NIST SP 800-122](https://csrc.nist.gov/publications/detail/sp/800-122/final) / [OWASP A02 Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)）
- マネージド DLP（[Datadog Sensitive Data Scanner](https://docs.datadoghq.com/sensitive_data_scanner/) / GCP DLP / Azure Purview）に出力前に通す選択肢も検討
- 派生 file（`trace.json` / `payload.json` / `gitleaks-report.json`）は処理後に `shred -u`（macOS は `rm -P`）で必ず破棄。共有 storage / git にコピーしない
- Datadog API: 本コードは Spans Events Search を使うが、テナントによっては log search（`/api/v2/logs/events/search`）に body が乗っている。field path（`.data[].attributes.custom.http.request.body` 等）は実テナントで `jq` で確認してから固定
- local の DB 状態が production と違うと再現できない → 本番 DB の sanitized dump を別途 restore
