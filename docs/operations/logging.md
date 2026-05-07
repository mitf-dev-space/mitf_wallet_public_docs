# Logging

Structured logging across all services (Users, Wallets, Ledger, Transactions, Customer Gateway, LoadTest.Job, Reconciliation.Job).

---

## Pipeline and outputs

- **Pipeline:** Serilog with shared `AddCustomLogger()`.
- **Console:** Always on — one JSON object per line (stdout) so `docker compose logs` and log shippers see structured fields.
- **Loki:** When `InternalLoggerOptions:LogType=Loki` and `ConnectionString` is set.
- **OTLP:** When `Observability:CollectorUrl` is set (logs sent to HTTP endpoint, typically port 4318).
- **SQL / PostgreSQL / Elasticsearch:** When `LogType` is `Sql`, `PostGres`, or `Elastic` and `ConnectionString` is set.
- **Console-only:** When `InternalLoggerOptions` is missing or `LogType` is `Console`.

---

## Standard fields

| Field | Description |
| ----- | ----------- |
| `Application` / `service.name` | Service identifier (e.g. `Masarat.Wallets.Api`) |
| `Environment` | Deployment environment |
| `trace_id` | OpenTelemetry trace ID |
| `span_id` | OpenTelemetry span ID |
| `CorrelationId` | Request-scoped ID; set in middleware; echoed in `X-Correlation-ID` response header |
| `LoadTestRunId` | LoadTest.Job: ID for the current run |
| `ReconciliationRunId` | Reconciliation job: ID for the current run |
| `Level` | Log level |
| `Exception` | Exception type and stack trace when present |

---

## Correlation ID

- **APIs:** Generated if not provided, or read from `X-Correlation-ID` request header.
- **Response:** `X-Correlation-ID` is returned so clients can use it for support.
- **Cross-service:** Send the same value in `X-Correlation-ID` on outbound calls; gRPC clients add metadata `x-correlation-id`.
- **Workers:** Use `LoadTestRunId` or `ReconciliationRunId` to filter all logs for a run.

---

## Query examples

### gRPC call logging

`GrpcLoggingInterceptor` logs each gRPC request at Information: **Method**, **StatusCode**, **DurationMs**.

```
gRPC call completed. Method: /wallet.WalletService/CreateWallet, Status: OK, DurationMs: 42
```

### Loki (LogQL)

```
{Application="Masarat.Wallets.Api"}
{Application=~"Masarat.*"} | json | CorrelationId="<guid>"
{Application=~"Masarat.*"} | json | trace_id="<trace-id>"
{Application="Masarat.LoadTest.Job"} | json | LoadTestRunId="<guid>"
```

### Elasticsearch (KQL)

```
Application: "Masarat.Wallets.Api"
CorrelationId: "<guid>"
trace_id: "<trace-id>"
```

---

## Configuration

### Observability (all services)

| Key | Description | Example |
| --- | ----------- | ------- |
| `Observability:ServiceName` | Service name in logs and traces | `Masarat.Wallets.Api` |
| `Observability:Environment` | Environment label | `Production` |
| `Observability:CollectorUrl` | OTLP collector URL | `http://otel-collector:4317` |

### InternalLoggerOptions

| Key | Description | Example |
| --- | ----------- | ------- |
| `InternalLoggerOptions:LogType` | `Console`, `Loki`, `Sql`, `PostGres`, `Elastic` | `Loki` |
| `InternalLoggerOptions:ConnectionString` | Required when LogType is not Console | `http://loki:3100` |
| `InternalLoggerOptions:TableName` | Table prefix for SQL sinks | `Logs` |

### Optional API request logging

| Key | Description | Default |
| --- | ----------- | ------- |
| `Observability:ApiLogging:EnableRequestLogging` | Log each request (method, path, status, duration) | `false` |
| `Observability:ApiLogging:LogRequestHeaders` | Include request headers (redacted per RedactHeaders) | `false` |
| `Observability:ApiLogging:LogRequestBody` | Include request body — not recommended in production | `false` |
| `Observability:ApiLogging:LogResponseBody` | Include response body — not recommended in production | `false` |
| `Observability:ApiLogging:RedactHeaders` | Header names to redact | `["Authorization","Cookie","X-Api-Key"]` |
| `Observability:ApiLogging:RedactBodyProperties` | JSON properties to redact | `["password","pin","apiKey","token","authorization"]` |
| `Observability:ApiLogging:ExcludePaths` | Paths excluded from request logging | `["/health","/health/ready","/metrics","/hc"]` |

For production: enable `EnableRequestLogging: true` with body logging off and keep default path exclusions.

### Log levels

- Default minimum: **Information**. Framework namespaces (Microsoft.*, System.*) overridden to **Warning**.
- Override per environment: `Logging:LogLevel:Default` in appsettings (e.g. `Debug` in Development).

---

## Runbook

### Error spikes

1. Filter by `Application` for the affected service.
2. Find `Level=Error` and note `CorrelationId` or `trace_id`.
3. Use that ID to see all log lines for the request across services.
4. Workers: filter by `LoadTestRunId` or `ReconciliationRunId`.

### Startup and shutdown

- **Started:** Each service logs once with `ServiceName`, `Environment`, `LogSink`, and `CollectorUrl` status.
- **Shutting down:** Each service logs once on stop. Use to confirm graceful shutdown ordering.

### Dependency failures

RabbitMQ and DB connection retries are logged with structured fields. Final failure is logged at Error. Search by `Application` and `Level=Error`.
