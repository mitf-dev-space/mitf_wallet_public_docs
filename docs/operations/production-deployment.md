# Production deployment

Sizing, ports, network, secrets, config. Audience: deployment, security, network.

---

## 1. Architecture Summary

- **APIs:** Ledger (5001), Wallets (5002), Users (5003), Transactions (5004), **Customer Gateway (5006)**; **Management Web (5007)**, **KYC API (5008)**. Optional: Webhooks (5005), Reconciliation.Api.
- **Workers:** **Masarat.LoadTest.Job** (idle by default; enable direct gRPC or **customer-gateway** load tests with `LoadTest__*` env vars and `compose/loadtest/*.yml` — see [load-testing-operations.md](load-testing-operations.md)), Reconciliation.Job, **Masarat.AmlBridge** (wallet completion event bridge to FlowGuard AML queue).
- **Data:** PostgreSQL 16 (multiple databases), RabbitMQ (AMQP + management).
- **Observability:** OTLP collector → Prometheus, Loki, Tempo → Grafana.
- **Optional:** Consul (configuration).

All APIs are .NET 10 (ASP.NET Core), expose gRPC (HTTP/2) and REST where applicable, and use MassTransit over RabbitMQ for internal messaging.

---

## 2. Server Specifications

### 2.1 Minimum Requirements (per environment)

Aligns with reference Compose limits.


| Component              | CPU  | Memory | Notes                                                                            |
| ---------------------- | ---- | ------ | -------------------------------------------------------------------------------- |
| **Ledger API**         | 0.5  | 512 MB | Double-entry and journal posting; can spike under load.                          |
| **Wallets API**        | 0.5  | 512 MB | Wallet CRUD, PIN, balance; calls Ledger over gRPC.                               |
| **Users API**          | 0.5  | 512 MB | Onboarding, user registration; calls Wallets over gRPC.                          |
| **Transactions API**   | 0.5  | 512 MB | Transfers, fund, merchant, withdrawal; calls Wallets + Ledger.                   |
| **Masarat.LoadTest.Job** | 0.5–2 | 512 MB–2 GB | Idle by default; load-test overlays may raise CPU/RAM. |
| **Reconciliation.Job** | 0.25 | 256 MB | Daily; exports ledger, matches statements.                                       |
| **PostgreSQL**         | 1+   | 2 GB+  | All DBs on one instance in reference setup; scale for data size and connections. |
| **RabbitMQ**           | 0.5  | 1 GB+  | Message broker; increase for high throughput.                                    |
| **OTLP Collector**     | 0.25 | 256 MB | Telemetry ingestion.                                                             |
| **Prometheus**         | 0.5  | 1 GB+  | Metrics retention depends on scrape interval and retention.                      |
| **Loki**               | 0.5  | 512 MB | Log retention and query load.                                                    |
| **Tempo**              | 0.25 | 512 MB | Trace storage.                                                                   |
| **Grafana**            | 0.25 | 256 MB | Dashboards and queries.                                                          |


### 2.2 Production Recommendations

- **APIs:** The current production reference baseline uses **4 vCPU and 4 GB RAM** per core API instance for the single-node strong-server profile validated by the latest customer baseline/peak runs. Scale horizontally behind a load balancer once sustained traffic outgrows a single strong node.
- **PostgreSQL:** Dedicated host or managed service; the current reference profile assumes roughly **6 vCPU and 8 GB RAM** with increased shared buffers and connection headroom.
- **RabbitMQ:** Persistent storage; consider clustering for HA; the current reference profile assumes **4 vCPU and 4 GB RAM** for the broker node.
- **Observability:** Separate nodes or managed services for Prometheus/Loki/Tempo to avoid contention with application workloads.
- **Disk:** SSD for PostgreSQL and RabbitMQ data; size for retention (metrics, logs, traces).

### 2.3 Container Runtime

- **Base image:** `mcr.microsoft.com/dotnet/aspnet:10.0` (APIs), `mcr.microsoft.com/dotnet/runtime:10.0` (workers).
- **Build image:** `mcr.microsoft.com/dotnet/sdk:10.0`.
- Health checks use `curl` in the container; ensure `curl` is available in the runtime image (reference Dockerfiles install it).

### 2.4 Reference stack parameters (repository defaults)

The repository now carries an **official production baseline** in service `appsettings.Production.json`, `docker-compose.yml`, and `docker-stack.yml`. Keep using environment-specific overrides for secrets, addresses, and for burst/stress-only capacity above this baseline.

| Layer                        | Parameter             | Reference value                                                   | Operational note                                                                                                                                                                                       |
| ---------------------------- | --------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **PostgreSQL**               | Version               | 16 (Alpine image in compose)                                      | Use a supported LTS or managed major aligned with migrations.                                                                                                                                          |
| **PostgreSQL**               | `max_connections`     | **1200**                                                          | Default `100` is too low when several .NET services each open an Npgsql pool; raising avoids Postgres error **53300** (*too many clients already*) under load. Increase with RAM/CPU; leave headroom for admins and migrations. |
| **PostgreSQL**               | Memory tuning         | `shared_buffers=2GB`, `work_mem=16MB`, `effective_cache_size=6GB` | Strong-server reference values validated in the latest customer production-style runs; tune further for your DB host and storage tier.                                                                   |
| **Npgsql (per process)**     | `Maximum Pool Size`   | **200** (Ledger, Wallets, Transactions, Users APIs)               | Cap **per replica** so the **sum over all services × replicas** stays **below** `max_connections` with margin.                                                                                         |
| **Npgsql**                   | Reconciliation job    | **20**                                                            | Lower ceiling for batch-style workload.                                                                                                                                                                |
| **Transactions API**         | Ingress backpressure (`TransferBackpressure__*`) | `Enabled=true`, `MaxInFlightTransfers=1024`, `AcquireTimeoutMs=500` | Shared cap on money-movement RPCs per process → **ResourceExhausted** when full ([transfer-backpressure client contract](../architecture/transfer-backpressure-client-contract.md)).                           |
| **Transactions API**         | Queued money RPCs     | (always on)                                                       | Transfer, fund, merchant/cash, pool create/fund, reverse **always** enqueue to RabbitMQ; scale **consumers** separately from ingress (see §10). Clients poll **GetRequestStatus** until terminal.                                                                               |
| **Gateway Customer API**     | Route-aware rate limits | `PermitLimit=2400`, `AuthBootstrap=1200` (login/refresh/onboarding only), `OperationStatus=4800`, `TransactionWrite=1800`, `Read=4800` (includes wallet provisioning + PIN verify) | Official stable production baseline. Keep more aggressive values in explicit peak/stress overlays instead of baking them into the default manifest. |
| **Users API**                | Onboarding rate limit | `PermitLimit=480`, `WindowSeconds=60`                             | Stable production baseline from the validated strong-server profile.                                                                                                                                    |
| **Ledger API**               | Backpressure          | `MaxInFlightRequests=256`, `MaxInFlightWriteRequests=192`, `MaxInFlightReadRequests=256`, timeouts `500/750/300 ms` | Protects the single-node ledger process from tail-latency collapse during heavy mixed traffic.                                                                                                         |
| **Wallets API**              | Messaging + ledger gRPC | `PrefetchCount=96`, `Set/Verify PIN=96`, `Change PIN=48`, ledger gRPC `192/192/128` | Stable production baseline; inbox cleanup remains disabled in this tuned profile to reduce churn during sustained pressure.                                                                             |
| **Transactions API**         | Messaging tuning      | `PrefetchCount=128`, `Transfer/Fund consumers=48`                 | Stable production baseline for the money-movement worker lanes.                                                                                                                                        |
| **API containers (production reference)** | CPU / memory limits   | **4 CPU, 4 GB** per API; **2 CPU, 2 GB** for the load-test worker | Strong-server reference profile used for the validated production-style runs.                                                                                                                            |

**Connection budget (illustrative):** with one instance each of Ledger, Wallets, Transactions, Users (200+200+200+200) plus Reconciliation (20), worst-case pool demand is already **820** connections before LoadTest/Webhooks/other clients. That is why the production baseline now carries **`max_connections=1200`**. If you add replicas or more worker processes, **recalculate** pools × replicas vs `max_connections` and either lower per-process pool ceilings or raise DB capacity.

---

## 3. Deployment Details

### 3.1 Port Matrix

**Host-exposed (e.g. load balancer or ingress):**


| Service                 | Port | Protocol                | Purpose                      |
| ----------------------- | ---- | ----------------------- | ---------------------------- |
| Ledger API              | 5001 | HTTP/1.1, HTTP/2 (gRPC) | Health, metrics, gRPC        |
| Wallets API             | 5002 | HTTP/1.1, HTTP/2 (gRPC) | Health, metrics, gRPC        |
| Users API               | 5003 | HTTP/1.1, HTTP/2        | Health, metrics, REST + gRPC |
| Transactions API        | 5004 | HTTP/1.1, HTTP/2 (gRPC) | Health, metrics, gRPC        |
| Customer Gateway API    | 5006 | HTTP/1.1                | Health, metrics, REST (orchestrates Users/Wallets/Transactions) |
| Webhooks API (optional) | 5005 | HTTP/1.1                | REST webhook management      |


**Internal (container) ports:**

Each API listens on two ports inside the container:

- **8080** — HTTP/1.1 (health `/health`, `/health/ready`, metrics `/metrics`, Swagger in Development).
- **8081** — HTTP/2 (gRPC only; used for service-to-service calls).

Reference Docker Compose maps **8080** to the host (5001–5004, 5006 for Gateway). gRPC between services uses **8081** on the Docker network (e.g. `masarat.ledger.api:8081`).

**Other HTTP apps in the repo:** Management portal (**5007**), KYC API (**5008**), and optional reporting/webhooks use the same container port pattern where applicable (HTTP on **8080** in Docker).

### 3.1.1 IIS (Windows Server) and Kestrel

The APIs and gateways avoid hardcoded `ListenLocalhost` when they detect IIS (`ASPNETCORE_PORT` from the ASP.NET Core Module, or `APP_POOL_ID`), so the reverse proxy can assign the process port. **Docker behaviour is unchanged** (`DOTNET_RUNNING_IN_CONTAINER` still uses 8080/8081 as above).

- **Configuration:** Continue to set connection strings, `RabbitMQ:*`, downstream gRPC addresses (`LedgerGrpc__Address`, `CustomerGateway__Downstream__*`, etc.), and `Auth:*` via environment variables, `web.config`, or transformed `appsettings.Production.json`—the same keys as in [Configuration reference](../reference/configuration-reference.md).
- **gRPC behind IIS:** Requires HTTP/2 (and usually TLS) at the site level; binding fixes in the app do not replace IIS manager configuration for HTTP/2 or certificates.

**Infrastructure:**


| Service             | Port(s)                  | Purpose                                |
| ------------------- | ------------------------ | -------------------------------------- |
| PostgreSQL          | 5432                     | Database                               |
| RabbitMQ            | 5672                     | AMQP                                   |
| RabbitMQ Management | 15672                    | Management UI (restrict in production) |
| Consul (optional)   | 8500, 8600/udp           | Config KV, DNS                         |
| OTLP Collector      | 4317 (gRPC), 4318 (HTTP) | Telemetry                              |
| Prometheus          | 9090                     | Metrics and scraping                   |
| Loki                | 3100                     | Logs                                   |
| Tempo               | 3200                     | Traces                                 |
| Grafana             | 3000                     | Dashboards                             |


### 3.2 Service Dependencies and Startup Order

1. **Infrastructure:** PostgreSQL, RabbitMQ (and optionally Consul) must be healthy first.
2. **Ledger API** — depends on DB + RabbitMQ.
3. **Wallets API** — depends on DB, RabbitMQ, Ledger API.
4. **Users API** — depends on DB, RabbitMQ, Wallets API.
5. **Transactions API** — depends on DB, RabbitMQ, Ledger API (and optionally Wallets for consistency).
6. **Customer Gateway API** — depends on Users, Wallets, Transactions APIs (HTTP + gRPC to downstreams).
7. **Masarat.LoadTest.Job** — depends on target APIs and optionally Gateway (see [load-testing-operations.md](load-testing-operations.md)).
8. **Reconciliation.Job** — depends on DB, Ledger API.

PostgreSQL init script creates: **MasaratLedger**, **MasaratWallets**, **MasaratUsers**, **MasaratReconciliation**.

### 3.3 Build and Run (reference)

- **Build:** From repo root, `docker compose build` (or `docker compose build --no-parallel` if NuGet timeouts occur).
- **Run:** `docker compose up -d`.
- **Health:** `GET http://<host>:5001/health`, `5002/health`, `5003/health`, `5004/health`, `5006/health` (Gateway) and `/health/ready` for readiness.

---

## 4. Network and Firewall

### 4.1 Recommended Segmentation

- **DMZ / edge:** Only the ports that clients and partners need: **5001–5006** (APIs; mobile apps typically use **5006** Gateway only) through a reverse proxy or load balancer. Do not expose **5432**, **5672**, **15672**, **4317/4318**, **9090**, **3100**, **3200**, **3000** to the public internet.
- **Application tier:** APIs and workers can reach:
  - PostgreSQL (5432)
  - RabbitMQ (5672; 15672 only for admin if required)
  - Each other (8080, 8081) for health and gRPC
  - OTLP Collector (4317 or 4318)
- **Backend tier:** PostgreSQL and RabbitMQ should accept connections only from the application tier and from management/deployment hosts as needed.
- **Observability:** Prometheus scrapes API metrics (8080); Loki/Tempo receive from the OTLP collector. Restrict access to Prometheus/Loki/Tempo/Grafana to operations and monitoring networks.

### 4.2 Firewall Checklist (production)


| Direction | Source           | Destination                       | Port                   | Purpose                                         |
| --------- | ---------------- | --------------------------------- | ---------------------- | ----------------------------------------------- |
| Inbound   | Clients / LB     | API hosts                         | 443                    | HTTPS (TLS termination)                         |
| Inbound   | LB / Ingress     | API containers                    | 5001–5005              | HTTP/gRPC (or 8080 if mapped)                   |
| Outbound  | APIs / Workers   | PostgreSQL                        | 5432                   | DB connections                                  |
| Outbound  | APIs / Workers   | RabbitMQ                          | 5672                   | AMQP                                            |
| Internal  | APIs             | Ledger/Wallets/Users/Transactions | 8080, 8081             | Health + gRPC                                   |
| Outbound  | APIs / Workers   | OTLP Collector                    | 4317, 4318             | Telemetry                                       |
| Inbound   | Prometheus       | APIs                              | 8080                   | /metrics scrape                                 |
| Inbound   | Ops / Monitoring | Grafana, Prometheus, Loki, Tempo  | 3000, 9090, 3100, 3200 | Dashboards and query (restrict to internal/VPN) |


### 4.3 TLS and HTTPS

- **Production:** Use **HTTPS/TLS** for all client-facing API traffic. Terminate TLS at the reverse proxy or load balancer; proxy to backend on 5001–5004 (or 8080) over a private network.
- **gRPC:** With TLS termination at the proxy, backend can remain HTTP/2 (h2c). If the proxy passes through TLS, ensure ALPN is configured for `h2` where gRPC is used.
- **Service-to-service:** In a trusted internal network, gRPC over plain HTTP/2 (e.g. `http://masarat.ledger.api:8081`) is acceptable; for higher assurance use mTLS or restrict by network and API key (see [system hardening](../security/system-hardening.md)).

---

## 5. Security Summary for Production

### 5.1 Secrets and Configuration

- **Do not commit:** API keys, `WalletAuthorizationToken:Secret`, database connection strings, RabbitMQ credentials.
- **Use:** Environment variables, secrets manager, or vault; override `appsettings` in production.
- **API key:** Set `Auth:ApiKey` and `Auth:RequireApiKey: true` on all APIs (see [system hardening](../security/system-hardening.md)).
- **Wallet PIN / transaction token:** Use a strong, shared `WalletAuthorizationToken:Secret` in Wallets and Transactions. Whether a debit requires a **VerifyWalletPin** token is determined per wallet **classification** (`OperationAuthMode`: user PIN vs external OTP trusted session), not by a Transactions config flag.

### 5.2 Authentication and Headers

- **REST / gRPC:** `X-Api-Key` or `Authorization: ApiKey <key>`; health endpoints `/health` and `/health/ready` are unauthenticated by design.
- **Bank context:** gRPC calls that act on behalf of a bank require metadata `x-bank-id: <bank-guid>` (authorization context; see [API reference](../reference/api.md)).
- **Swagger / OpenAPI:** Disable or restrict in production to avoid exposing API structure.

### 5.3 Logging and PII

- **Request/response logging:** Disable body logging in production (`LogRequestBody`, `LogResponseBody: false`).
- **Redact:** Sensitive headers (`Authorization`, `Cookie`, `X-Api-Key`) are redacted when request logging is enabled (see [system hardening](../security/system-hardening.md) and [logging](logging.md)).

### 5.4 Operational Hardening


| Area    | Recommendation                                                         |
| ------- | ---------------------------------------------------------------------- |
| TLS     | HTTPS for all client-facing traffic; terminate at proxy.               |
| Secrets | Vault or env vars; never in source control.                            |
| API key | Strong key; `RequireApiKey: true` on all APIs.                         |
| Health  | Use `/health` and `/health/ready` for probes only; no sensitive logic. |
| Network | Restrict DB, RabbitMQ, and observability to required hosts and ports.  |
| Swagger | Disable or restrict in production.                                     |


---

## 6. Configuration Reference (Production)

### 6.1 Connection Strings (PostgreSQL)


| Service            | Config key                          | Database              |
| ------------------ | ----------------------------------- | --------------------- |
| Ledger API         | `ConnectionStrings__Ledger`         | MasaratLedger         |
| Wallets API        | `ConnectionStrings__Wallets`        | MasaratWallets        |
| Users API          | `ConnectionStrings__Users`          | MasaratUsers          |
| Transactions API   | `ConnectionStrings__Transactions`   | `masarattransactions` |
| Reconciliation.Job | `ConnectionStrings__Reconciliation` | MasaratReconciliation |


Use a dedicated user per service and strong passwords; ensure DB exists (init script or migration).

### 6.2 gRPC Client Addresses (service-to-service)


| Consumer           | Config                                                                                         | Target service                                       |
| ------------------ | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Wallets API        | `LedgerGrpc__Address`                                                                          | Ledger API (e.g. `http://masarat.ledger.api:8081`)   |
| Transactions API   | `LedgerGrpc__Address`                                                                          | Ledger API                                           |
| Transactions API   | `WalletGrpc__Address`                                                                          | Wallets API (e.g. `http://masarat.wallets.api:8081`) |
| Users API          | `WalletGrpc__Address`                                                                          | Wallets API                                          |
| Masarat.LoadTest.Job | `LoadTest__UserServiceAddress`, `LoadTest__WalletServiceAddress`, `LoadTest__TransactionServiceAddress`; optional `LoadTest__CustomerGatewayLoadTest__BaseUrl` | Users, Wallets, Transactions (gRPC); Gateway (HTTP) when gateway load test is enabled |
| Reconciliation.Job | `Reconciliation__LedgerGrpcAddress`                                                            | Ledger API (gRPC port)                               |


### 6.3 RabbitMQ

- **Config section:** `RabbitMQ__Host`, `RabbitMQ__Port`, `RabbitMQ__Username`, `RabbitMQ__Password`.
- All APIs and workers that use MassTransit need the same broker; use a dedicated vhost and user per environment in production.
- **AML bridge routing config:** `AmlIntegration__Enabled`, `AmlIntegration__FlowGuardExchangeName`, `AmlIntegration__FlowGuardRoutingKeyTemplate`, and `AmlIntegration__BankCodes__<bank-guid>=<BANKCODE>`.
- **AML bridge hardened broker config:** `AmlIntegration__RabbitMq__Host`, `Port`, `VirtualHost`, `Username`, `Password`, `UseSsl`, `SslServerName`.
- **Least privilege:** bridge user should only have required publish/consume permissions for wallet completion and AML routing paths; do not reuse admin credentials.

### 6.4 Observability

- **OTLP:** `Observability__CollectorUrl` (e.g. `http://otelcollector:4317`), `Observability__ServiceName`, `Observability__Environment`.
- **Loki (optional):** `InternalLoggerOptions__ConnectionString` (e.g. `http://loki:3100`).
- **Prometheus:** APIs expose `/metrics` on the HTTP port (8080); Prometheus scrapes these targets (see `infra/prometheus.yml`).

### 6.5 Transactions API (fees and settlement)

- **Fees:** `Fees__FeeRevenueAccountId`, `Fees__MerchantSettlementAccountId`, `Fees__CashSettlementAccountId` (ledger account GUIDs).
- **Idempotency:** `Idempotency__TtlHours` (default 24).

### 6.6 Optional: Consul

- **Config source:** `ConfigSource:Type=consul`, `CONSUL_ADDRESS`, `CONSUL_CONFIG_KEY` for centralised configuration.

### 6.7 Masarat.LoadTest.Job

Reference compose: **`LoadTest__LoadTest__Enabled=false`** and **`LoadTest__CustomerGatewayLoadTest__Enabled=false`** keep the worker idle. Enable direct gRPC load tests and/or **customer-gateway** journeys from `compose/loadtest/*.yml`; keep secrets (e.g. signing keys, gateway API keys) in a vault, not in git. See [load-testing-operations.md](load-testing-operations.md).

---

## 7. Health Checks and Readiness

- **Endpoints:** `GET /health`, `GET /health/ready` on each API (HTTP port, e.g. 8080).
- **Behaviour:** Return 200 when the service and its dependencies (DB, RabbitMQ) are healthy; no API key required.
- **Orchestration:** Use `/health/ready` for Kubernetes readiness and load balancer registration; use `/health` for liveness if needed.

---

## 8. Database Layout

Single PostgreSQL instance (reference) with multiple databases:


| Database              | Used by                       | Purpose                                                                               |
| --------------------- | ----------------------------- | ------------------------------------------------------------------------------------- |
| MasaratLedger         | Ledger API                    | Accounts, entries, balances                                                           |
| MasaratWallets        | Wallets API, Transactions API | Wallets, classifications, fee rules, PIN data; Transaction records (Transactions API) |
| MasaratUsers          | Users API                     | Users (resident/foreign), onboarding                                                  |
| MasaratReconciliation | Reconciliation.Job            | Reconciliation runs, exceptions                                                       |


Init script: `infra/init-db.sh` (creates databases when `POSTGRES_MULTIPLE_DATABASES` is set).

---

## 9. Deployment Checklist

**Deployment team**

- Build images from tagged source; use non-root user and read-only filesystem where possible.
- Set `ASPNETCORE_ENVIRONMENT=Production` (or equivalent) for APIs.
- Configure connection strings, RabbitMQ, and gRPC addresses for the production network.
- Map host ports or ingress to API HTTP/gRPC ports; ensure 8081 is reachable between services if gRPC is on a separate port.
- Run DB init or migrations; verify all four databases exist.
- Confirm health and readiness probes use `/health` and `/health/ready`.
- Configure OTLP collector URL and optional Loki; verify Prometheus scrape targets.

**Security team**

- Enable API key auth (`Auth:RequireApiKey: true`, strong `Auth:ApiKey`) on all APIs.
- Store secrets in a vault or env vars; no secrets in image or config in repo.
- Use strong `WalletAuthorizationToken:Secret` in Wallets and Transactions; align wallet **classifications** (`OperationAuthMode`) with whether PIN step-up is required for debits.
- Disable or restrict Swagger/OpenAPI in production.
- Ensure request/response body logging is disabled; sensitive headers redacted (see [system hardening](../security/system-hardening.md)).

**Network team**

- Expose only API ports (e.g. 443 → 5001–5005) to clients; TLS termination at proxy/LB.
- Restrict PostgreSQL (5432) and RabbitMQ (5672, 15672) to application and management hosts.
- Allow internal gRPC (e.g. 8081) only between API/worker hosts and Ledger/Wallets/Users/Transactions.
- Restrict Prometheus, Loki, Tempo, Grafana to operations/monitoring network or VPN.
- Document and implement firewall rules per section 4.2.

---

## 10. Load and capacity

Flow: clients → **Transactions** → **Wallets** + **Ledger** + **Postgres** + **RabbitMQ**. Bottlenecks are usually DB round-trips and consumer throughput, not CPU. Numbers: measure in your environment and compare against the current reference runs.

| Control | Role |
|--------|------|
| `TransferBackpressure__*` | Shared cap on in-flight **money-movement** RPCs per process (Transfer, FundWallet, merchant/cash, pool, reverse, etc.) → **ResourceExhausted**; clients: same idempotency key ([transfer-backpressure client contract](../architecture/transfer-backpressure-client-contract.md)) |
| Npgsql `Maximum Pool Size` | Stops connection storms |
| `max_connections` | Must exceed Σ pools × replicas + headroom |
| `Messaging:Tuning` / worker replicas | Consumer throughput, prefetch, concurrency; watch queue depth |

**Scaling:** DB lock/wait → tune DB/ledger before adding API replicas (more replicas can add contention). Async backlog → add **worker** replicas; recheck pools/`max_connections`. **53300** → raise `max_connections` or lower pool sizes. Reference compose: main Transactions service is not `scale`’d; extra consumers are modeled as separate worker services.

**Sample throughput (Docker, not SLA):** see **[load test reference runs](../load-testing/load-test-reference-runs.md)** — recent pair: **~146 ops/s** (10k no chaos) and **~89 ops/s** (10k chaos), **ReplayMismatches=0** on chaos. **Available** balance can lag **ledger** under async, so reconciliation and consistency checks should prefer ledger-aligned balances.

**Checklist:** Document pools vs `max_connections`; share backpressure contract with integrators; workers + **GetRequestStatus** polling for integrators; alert on connections, queue depth, errors, p95.

---

## 11. Related docs

- [Configuration reference](../reference/configuration-reference.md)
- [System hardening](../security/system-hardening.md)
- [Logging](logging.md)
- [Reconciliation & consistency runbook](reconciliation-and-consistency-runbook.md)
- [Transfer backpressure](../architecture/transfer-backpressure-client-contract.md)
- [Load test reference runs](../load-testing/load-test-reference-runs.md)


