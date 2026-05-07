# Configuration reference

JSON sections, env var overrides, and per-service configuration for all runnable applications.

**Convention — environment variables:** ASP.NET Core maps nested JSON to env vars with `__` (double underscore), e.g. `ConnectionStrings__Ledger`, `LedgerGrpc__Address`, `CustomerGateway__Downstream__UsersBaseUrl`.

**Files:** Each API typically ships `appsettings.json` plus optional `appsettings.{Environment}.json` (e.g. **Development** relaxes API keys). **Production** overlays in the repo tune backpressure and messaging without duplicating secrets.

**Optional Consul:** Set `CONFIG_SOURCE=consul` with `CONSUL_ADDRESS` / `CONSUL_CONFIG_KEY` to load settings from Consul KV (see Observability/Consul integration in code).

**Config source (JSON):** `ConfigSource:Type` is set to `local` in shipped `appsettings.json` so operators see explicitly that configuration comes from files unless overridden by `CONFIG_SOURCE` (Consul) or environment variables. It is informational for the Consul bootstrap path in code, not a separate product feature.

**Kestrel and hosting (web APIs / gateways):** Each host uses shared logic in `Masarat.ApiCommon.Hosting` (`ConfigureMasaratKestrel`):

| Deployment | Behaviour |
| ---------- | --------- |
| **Docker** (`DOTNET_RUNNING_IN_CONTAINER`) | Unchanged: **8080** (HTTP/1 — health, metrics) and **8081** (HTTP/2 — gRPC) for core APIs; gateways listen on **8080** only. |
| **Self-hosted Kestrel** (Linux/macOS/Windows console) | If `Kestrel:Endpoints` is present in configuration, Kestrel binds using the standard ASP.NET Core `Kestrel` config schema (`Url`, `Protocols` per endpoint). Default repo JSON matches the historical localhost ports (5001–5004, 5006–5008). |
| **IIS** (out-of-process ANCM: `ASPNETCORE_PORT`, or worker `APP_POOL_ID`) | The app **does not** call `Listen*` in code so the ASP.NET Core Module can assign the backend port. **Do not** rely on `Kestrel:Endpoints` in appsettings for the IIS site binding—configure the site and forwarding in IIS; use appsettings/env for connection strings, RabbitMQ, downstream gRPC URLs, and auth. |

**Shared telemetry (most APIs):**

| Section | Purpose |
| -------- | -------- |
| `Observability` | `ServiceName`, `Environment`, `CollectorUrl` (OTLP gRPC), `PrometheusEndpoint`, `EnableTracing`, `EnableMetrics`, `EnableMassTransitInstrumentation`, `EnableHttpClientInstrumentation` |
| `InternalLoggerOptions` | Serilog sinks: `LogType` (`Console`, `Loki`, etc.), `ConnectionString`, `TableName`; worker apps may use `ConsoleFormat` |

**Shared messaging (services using RabbitMQ):**

| Section | Purpose |
| -------- | -------- |
| `RabbitMQ` | `Host`, `Port` (optional), `Username`, `Password` |
| `Messaging:Tuning` | See per-service tables below (prefetch, retries, consumer limits, outbox tuning) |

For hardened bridge deployments, `AmlIntegration:RabbitMq` adds `VirtualHost`, `UseSsl`, and `SslServerName`.

**Shared API security (`Masarat.ApiCommon`):**

| Section | Purpose |
| -------- | -------- |
| `Auth` | `ApiKey` — shared secret; `RequireApiKey` — when `true`, all routes except `/health` and `/health/ready` require the key (REST header or gRPC metadata). Often **disabled** in `appsettings.Development.json` for local dev. |

---

## 1. Masarat.Ledger.Api

**Project:** `src/Masarat.Ledger/Masarat.Ledger.Api`  
**Database:** `ConnectionStrings:Ledger` → PostgreSQL `MasaratLedger`.

| Section / key | Description |
| --------------- | ------------- |
| `Ledger` | `AllowedCurrencies` — optional allow-list (e.g. `LYD`, `USD`); empty/null allows any 3-letter code. `DeferredInlineSnapshotAccountIds` — ledger account GUIDs that skip **inline** balance snapshot updates in `PostJournal` (hot system accounts). |
| `LedgerBackpressure` | `Enabled`, `MaxInFlightRequests`, `AcquireTimeoutMs`; optional `MaxInFlightReadRequests`, `MaxInFlightWriteRequests`, `ReadAcquireTimeoutMs`, `WriteAcquireTimeoutMs`. Protects the ledger process from overload ([Transfer backpressure client contract](../architecture/transfer-backpressure-client-contract.md)). |
| `Messaging:Tuning` | `PrefetchCount`, `RetryCount`, `RetryIntervalSeconds` for MassTransit consumers. |
| `HandlerTiming` | `Enabled`, `MinDurationMs`, `LogFailuresAlways` — optional handler duration diagnostics (`LedgerHandlerTimingOptions`). |

RabbitMQ is used for **consumers** (e.g. completion events → snapshots), not for the primary synchronous gRPC path.

---

## 2. Masarat.Wallets.Api

**Project:** `src/Masarat.Wallets/Masarat.Wallets.Api`  
**Database:** `ConnectionStrings:Wallets` → PostgreSQL `MasaratWallets` (include `Maximum Pool Size` in production).

| Section / key | Description |
| --------------- | ------------- |
| `LedgerGrpc` | `Address` — Ledger gRPC base URL (**HTTP/2**, typically port **8081** in Docker). Optional client-side throttling: `MaxInFlightRequests`, `AcquireTimeoutMs`, and optional read/write splits (`MaxInFlightReadRequests`, `MaxInFlightWriteRequests`, `ReadAcquireTimeoutMs`, `WriteAcquireTimeoutMs`). |
| `Idempotency` | `RetentionDays` — how long to keep idempotency rows before cleanup (0 disables cleanup). `TtlHours` — legacy; converted to days if `RetentionDays` unset. |
| `WalletPin` | `MaxFailedAttempts`, `LockoutMinutes`, `MinPinLength`, `MaxPinLength`. |
| `WalletAuthorizationToken` | `Secret` — **must match** Transactions. `ExpiryMinutes` — lifetime of debit authorization tokens issued after `VerifyWalletPin`. |
| `Fees` | `FeeRevenueAccountId`, `MerchantSettlementAccountId`, `CashSettlementAccountId` — optional in this service; **Transactions** is the primary owner of settlement GUIDs for money movement. |
| `Messaging:Tuning` | Standard prefetch/retry **plus** Wallets-specific: `SetWalletPinConsumerConcurrentLimit`, `VerifyWalletPinConsumerConcurrentLimit`, `ChangeWalletPinConsumerConcurrentLimit`. **Outbox (EF):** `OutboxIsolationLevel`, `OutboxDuplicateDetectionWindowMinutes`, `OutboxQueryDelayMs`, `OutboxQueryMessageLimit`, `OutboxMessageDeliveryLimit`, `OutboxDisableDeliveryService`, `OutboxDisableInboxCleanupService`. |
| `OrphanedLedgerDetection` | `InitialDelaySeconds`, `ScanIntervalMinutes`, `LookbackHours`, `LedgerQueryLimit` — **orphaned ledger account** background scan (ledger accounts without local wallet rows). |

MassTransit uses the **EF transactional outbox** on `WalletsDbContext` so publishes commit with wallet state.

---

## 3. Masarat.Users.Api

**Project:** `src/Masarat.Users/Masarat.Users.Api`  
**Database:** `ConnectionStrings:Users` → PostgreSQL `MasaratUsers`.

| Section / key | Description |
| --------------- | ------------- |
| `WalletGrpc` | `Address` — Wallets service gRPC URL (HTTP/2). |
| `Messaging:Tuning` | `PrefetchCount`, `RetryCount`, `RetryIntervalSeconds`. |
| `OnboardingRateLimiting` | `PermitLimit`, `WindowSeconds` — rate limit for **POST `/onboarding/accounts`**. |

---

## 4. Masarat.Transactions.Api

**Project:** `src/Masarat.Transactions/Masarat.Transactions.Api`  
**Database:** `ConnectionStrings:Wallets` → same PostgreSQL DB as Wallets (`MasaratWallets`) for transactions and shared reads.

| Section / key | Description |
| --------------- | ------------- |
| `LedgerGrpc` | Same shape as Wallets (`Address`, optional in-flight limits). |
| `Idempotency` | Same `Idempotency` options as Wallets (`RetentionDays`, `TtlHours`) — used by cleanup service. |
| `Maintenance` | `PendingRepairEnabled`, `PendingRepairScanIntervalSeconds`, `PendingRepairAgeSeconds`, `PendingRepairBatchSize`, `PendingRepairMaxBatchesPerScan` (catch-up batches per wake) — **pending transaction repair** loop. `IdempotencyCleanupEnabled`, `IdempotencyCleanupIntervalMinutes` — periodic idempotency row deletion. |
| `WalletAuthorizationToken` | `Secret`, `ExpiryMinutes` — **must match Wallets** for token validation. |
| `OperationAuthMode` (classification) | Not in Transactions JSON — stored on each wallet **classification** in Wallets. **User PIN** → debit RPCs require `transaction_authorization_token` from **VerifyWalletPin**; **External OTP trusted session** → token not required. Set via Wallets / management UI. |
| `TransferBackpressure` | `Enabled`, `MaxInFlightTransfers`, `AcquireTimeoutMs` — caps concurrent **synchronous** transfer (and related) RPC acceptance before enqueue. |
| `HandlerTiming` | `Enabled`, `MinDurationMs`, `LogFailuresAlways` — command handler timing logs. |
| `Fees` | `FeeRevenueAccountId`, `MerchantSettlementAccountId`, `CashSettlementAccountId` — **ledger account GUIDs** for fees and settlement (required for merchant/cash flows). |
| `Messaging:Tuning` | `PrefetchCount`, `RetryCount`, `RetryIntervalSeconds`, `TransferConsumerConcurrentLimit`, `FundWalletConsumerConcurrentLimit`. **EF outbox (same keys as Wallets):** `OutboxIsolationLevel`, `OutboxDuplicateDetectionWindowMinutes`, `OutboxQueryDelayMs`, `OutboxQueryMessageLimit`, `OutboxMessageDeliveryLimit`, `OutboxDisableDeliveryService`, `OutboxDisableInboxCleanupService`. |

MassTransit **EF outbox** on `TransactionsDbContext`; consumers process queued transfer/fund/merchant/cash/pool/reverse commands. Money RPCs enqueue to RabbitMQ and clients poll for async status per product design.

---

## 5. Masarat.Gateway.Customer.Api

**Project:** `src/Masarat.Gateway/Masarat.Gateway.Customer.Api`  
**Database:** none (orchestration only).

| Section / key | Description |
| --------------- | ------------- |
| `Auth` | `ApiKey` / `RequireApiKey` — optional **global** gateway API key via `ApiKeyOptions` (if used alongside app keys; see code). |
| `CustomerGateway:RequireUserContextForWrites` | User JWT required for mutating routes when true. |
| `CustomerGateway:RequireWalletPinForLogin` | When `true` (default), login verifies the wallet PIN whenever the wallet already has a PIN. **First PIN** is set via optional **`pin`** on **`POST .../onboarding/accounts`** (gateway calls Wallets with a trusted actor after provisioning)—there is no separate customer-gateway **set first PIN** route. When `false`, login can skip PIN until a PIN exists; first PIN still uses onboarding (or authenticated **change** flows as applicable). |
| `CustomerGateway:OnboardingWalletPin` | `MinLength`, `MaxLength` — gateway-side validation for optional onboarding `pin` (keep aligned with Wallets `WalletPin`). |
| `CustomerGateway:AllowUserIdHeaderFallback` | **Dangerous** in production — allows inferring user without JWT when true; keep `false` in prod. |
| `CustomerGateway:EnableBusinessPooledAccounts` | Feature flag for business pooled-account routes. |
| `CustomerGateway:EnableBusinessSubWallets` | Feature flag for sub-wallet routes. |
| `CustomerGateway:EnableMerchantAcceptanceRoutes` | Feature flag for merchant acceptance flows. |
| `CustomerGateway:OnboardingProvisioningPollAttempts` / `OnboardingProvisioningPollDelayMs` | Polling after onboarding until wallet is ready. |
| `CustomerGateway:UserAuthentication` | `Enabled`, `Issuer`, `Audience`, `SigningKey`, `AccessTokenExpiryMinutes`, `RefreshTokenExpiryDays`, `RequireHttpsMetadata`. |
| `CustomerGateway:DevelopmentAuthOverride` | **Development only:** fixed bearer (`BearerToken`), synthetic user fields, optional `DefaultAppId` / `DefaultAppApiKey` so Swagger can omit app headers. |
| `CustomerGateway:RateLimiting` | Global `PermitLimit`, `WindowSeconds`, `QueueLimit`. **Partitioned:** `AuthBootstrap`, `OperationStatus`, `TransactionWrite`, `Read` — each may set `PermitLimit`, `WindowSeconds`, `QueueLimit`. Production overrides often raise these (see `appsettings.Production.json`). |
| `CustomerGateway:Downstream` | `UsersBaseUrl` (HTTP), `UsersGrpcAddress`, `WalletGrpcAddress`, `TransactionsGrpcAddress` — base URLs for outbound calls. `DownstreamApiKey` — optional single key sent to **all** core APIs when configured. |
| `CustomerGateway:DownstreamResilience` | (Bound from code defaults if not in JSON) `ReadTimeoutMs`, `WriteTimeoutMs`, `HttpWriteTimeoutMs`, `MaxRetryAttempts`, `BaseDelayMs`, `BreakDurationSeconds`, `MinimumThroughput`, `FailureRatio` — Polly-style resilience for downstream calls. |
| `CustomerGateway:Apps` | Array of `AppId`, `AppType` (`customer` / `business` / `merchant`), `BankId`, `ApiKey`, `Enabled` — **per-app** credentials and tenant binding. Clients send `X-App-Id` + app key (headers) per gateway contract. |

HTTP binding follows the same **Kestrel / IIS** rules as the other web hosts (see table at the top of this page). For self-hosted runs, `Kestrel:Endpoints` in `appsettings.json` defaults to **http://127.0.0.1:5006** with HTTP/1.

---

## 6. Masarat.LoadTest.Job

**Project:** `src/Masarat.LoadTest/Masarat.LoadTest.Job`  
**Worker host** — no public HTTP API.

| Section / key | Description |
| --------------- | ------------- |
| `LoadTest:UserServiceAddress` / `WalletServiceAddress` / `TransactionServiceAddress` | gRPC addresses (HTTP/2) for **direct** API load tests. |
| `LoadTest:BankId` | Sent as `x-bank-id` on gRPC metadata. |
| `LoadTest:FundAmount`, `TransferAmount`, `Currency` | Demo journey amounts. |
| `LoadTest:CashWithdrawalAmount`, `MerchantPaymentAmount`, `MerchantReference` | Optional scenario steps. |
| `LoadTest:TestPin` | If set, exercises PIN flows on first wallet. |
| `LoadTest:CreatePooledAccount`, `FundFromPoolAmount`, `ReverseLastTransaction` | Toggle pooled-account and reversal steps. |
| `LoadTest:Employer`, `SupWalletCustomers`, `EligibleCustomers` | Seeded customer data for provisioning journeys. |
| `LoadTest:LoadTest` | **Direct gRPC stress:** `Enabled`, `WalletCount`, `TransferCount`, `WarmupWalletCount`, `SoakRounds`, `SoakDelaySeconds`, `MaxConcurrency`, `ProvisionConcurrency`, `FundingConcurrency`, `TransferAmount`, `FundEachSourceWallet`, `AuthorizationTokenSecret` (must align with Wallets/Transactions token secret when PIN paths run), `Chaos` (* rates, delays, consistency sample size), `TransferSlo` (thresholds, `FailOnBreach`). |
| `LoadTest:CustomerGatewayLoadTest` | **HTTP gateway** journeys: `Enabled`, `BaseUrl`, `GatewayApiKey`, `Profile`, `Mode`, `Persona`, `PersonaSequence`, `VirtualUserCount`, `MaxConcurrency`, `IterationsPerUser`, think times, polling and 429-retry settings, HTTP client limits, provisioning flags, sub-persona weights, and per-persona blocks `Customer`, `Business`, `Merchant` (`AppId`, `AppKey`, `BankId`, naming prefixes, counts, etc.). See `LoadTestServiceOptions.cs` and `appsettings.json` for the full shape. |

Observability: `Observability:CollectorUrl`, `ServiceName`, `Environment`; logging via `InternalLoggerOptions`.

---

## 7. Masarat.Reconciliation.Job

**Project:** `src/Masarat.Reconciliation/Masarat.Reconciliation.Job`  
**Databases:** `ConnectionStrings:Reconciliation` (reconciliation tables), `ConnectionStrings:Wallets` (read **Transactions** EF context for internal consistency).

| Section / key | Description |
| --------------- | ------------- |
| `Reconciliation:LedgerGrpcAddress` | Ledger gRPC for `ExportEntries`. |
| `Reconciliation:RunAtUtcHour` | Scheduled wake hour (UTC, 0–23). Out-of-range values are clamped; logged at warning. |
| `Reconciliation:MaxCatchUpDays` | Calendar days back (including yesterday) scanned each wake for dates without a **Completed** run (minimum effective 1). Default `30`. |
| `Reconciliation:AbandonedRunAgeHours` | A **Running** row older than this many hours is removed before retry; younger **Running** rows block a duplicate attempt. `0` disables automatic removal of stuck **Running** rows (failed rows are still removed on retry). Default `2`. |
| `Reconciliation:BankAmountMatchTolerance` | Absolute amount difference allowed when matching ledger vs bank (same currency); `0` means exact equality. |
| `Reconciliation:MockBank` | `Entries`, `IncludeUndatedEntries` — mock bank lines; dated entries filter to the reconciliation `runDate` (UTC calendar day). |
| `InternalConsistency` | `Enabled`, `PollIntervalSeconds`, `BatchSize` — internal consistency worker between local state and ledger. |

Also uses `Observability` / `InternalLoggerOptions` like other hosts.

---

## 8. Masarat.Reconciliation.Reporting

**Project:** `src/Masarat.Reconciliation/Masarat.Reconciliation.Reporting`  
Minimal API — reporting only, no DB connection string in default `appsettings`; mappings are file-based.

| Section / key | Description |
| --------------- | ------------- |
| `Reporting:TransactionsGrpcAddress` | Transactions service gRPC for exports. |
| `Reporting:LedgerGrpcAddress` | Ledger gRPC. |
| `Reporting:ExportPageSize`, `MaxExportPages` | Paging guards for large exports. When the server stops paging after `MaxExportPages` but more data exists, it returns partial results with `X-Masarat-Export-Truncated: true` (and `X-Masarat-Export-Scope: bank-facing-settlement`) instead of throwing. |
| `BankAccountMappings:Accounts` | List of `BankId`, `SettlementType`, `AccountKey`, `BankAccountNumber`, `BankName` (settlement account label in reports), and optional `BankDisplayName` (institution label for management bank pickers). |

**HTTP:** Aggregated settlement endpoints set `X-Masarat-Export-Scope` / `X-Masarat-Export-Truncated`. Single-page `/api/reports/transactions` includes `exportSource` in JSON and `X-Masarat-Export-Truncated: false` when only one page is returned.

---

## 9. Masarat.Webhooks.Api

**Project:** `src/Masarat.Webhooks/Masarat.Webhooks.Api`  
**Database:** `ConnectionStrings:Webhooks` → PostgreSQL `MasaratWebhooks` (falls back to localhost in code if unset).

| Key | Description |
| --- | ----------- |
| `RabbitMQ:*` | Broker connection (same as other services). |
| MassTransit | `PrefetchCount` and retry interval are **code defaults** in `Program.cs` (10 prefetch, 3 retries × 5 s) unless extended to configuration later. |

No `appsettings.json` in repo for this project — configure via env vars or add JSON.

---

## 10. Masarat.AmlBridge

**Project:** `src/Masarat.AmlBridge`  
**Databases:** `ConnectionStrings:Transactions` (`masarattransactions`) and `ConnectionStrings:Wallets` (`MasaratWallets`) for tenant resolution.

| Section / key | Description |
| --------------- | ------------- |
| `AmlIntegration:Enabled` | Kill-switch to disable bridge publishing without removing consumers. |
| `AmlIntegration:FlowGuardExchangeName` | AML topic exchange (default `aml.transactions`). |
| `AmlIntegration:FlowGuardRoutingKeyTemplate` | Routing template with `{BankCode}` (default `transaction.{BankCode}`). |
| `AmlIntegration:BankCodes` | Mapping `BankId GUID -> BankCode` (env form `AmlIntegration__BankCodes__<guid>`). |
| `AmlIntegration:RabbitMq` | Bridge broker connection: `Host`, `Port`, `VirtualHost`, `Username`, `Password`, `UseSsl`, `SslServerName`. |
| `RabbitMQ` | Legacy fallback section for host/port/credentials if `AmlIntegration:RabbitMq` is not set. |

Use a dedicated RabbitMQ user/vhost in production and enable TLS (`UseSsl=true`) with server name validation.

---

## Quick cross-service checklist

| Concern | Services |
| -------- | -------- |
| **Ledger DB** | Ledger API |
| **Wallets / transactions DB** | Wallets API, Transactions API |
| **Users DB** | Users API |
| **Reconciliation DB** | Reconciliation.Job |
| **Webhooks DB** | Webhooks.Api |
| **Bridge DB reads** | AmlBridge (Transactions + Wallets) |
| **Matching `WalletAuthorizationToken:Secret`** | Wallets API, Transactions API |
| **Settlement ledger GUIDs** | Primarily **Transactions** `Fees:*` |
| **Ledger gRPC address** | Wallets, Transactions, Reconciliation.Job, Reconciliation.Reporting |
| **Outbox** | Wallets, Transactions |

---

See also: [Production deployment](../operations/production-deployment.md) · [System hardening](../security/system-hardening.md) · [Logging](../operations/logging.md) · [Platform capabilities](../architecture/platform-capabilities.md)
