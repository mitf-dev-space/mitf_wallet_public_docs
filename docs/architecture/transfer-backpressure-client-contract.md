# Transaction ingress backpressure — client contract

*(Config section name `TransferBackpressure` retained for compatibility.)*

When money-moving RPCs exceed the per-process concurrency cap, the API fails fast with **`ResourceExhausted`**. Code: `TransferBackpressureOptions`, `TransferConcurrencyGate`, `TransactionIngressBackpressureInterceptor`.

**On `ResourceExhausted`:** retry with same **idempotency key** + backoff + jitter.

## Gated RPCs (shared limit)

One pool counts **all** of these together (not per method):

`Transfer`, `FundWallet`, `ProcessMerchantPayment`, `ProcessCashWithdrawal`, `FundWalletFromPooledAccount`, `CreatePooledAccount`, `ReverseTransaction`

**Not gated:** `GetRequestStatus`, `GetTransaction`, `ListTransactions`, `GetPooledAccount`, `ListPooledAccounts`, `GetSpendingSummary`

## Config

| Setting | Meaning |
| ------- | ------- |
| `TransferBackpressure:Enabled` | Gate on/off |
| `TransferBackpressure:MaxInFlightTransfers` | Max concurrent gated RPCs per API process (shared across listed methods) |
| `TransferBackpressure:AcquireTimeoutMs` | Wait for a slot (ms); `0` = immediate reject |

Per replica: ≈ `MaxInFlightTransfers × replica count` before DB/ledger limits. Defaults: [Production deployment](../operations/production-deployment.md) (§2.4).

## gRPC error handling

| Status | Client action |
| ------ | ------------- |
| **ResourceExhausted** | Retryable. Backoff + jitter. Same idempotency key. |
| **FailedPrecondition** | Limited retries + same key; then ops + logs (`DiagnosticSummary`). Some cases are validation, not transient. |
| **DeadlineExceeded** | Retry with same key only if product accepts idempotent retry; else treat as unknown and reconcile via history/status. |

## Queued money RPCs

Gated RPCs **always** enqueue: they may return **`QUEUED` / `PROCESSING`** and `request_id`. Poll **`GetRequestStatus`** until terminal before treating funds as moved. **FAILED** → retry with same idempotency key where applicable. Supplying your own `idempotency_key` is preferred for cross-retry correlation.

**Spikes:** Some `ResourceExhausted` is expected under reject-at-capacity. SLOs should state whether rejection rate counts.

## Code

- `src/Masarat.Transactions/Masarat.Transactions.Api/TransferBackpressureOptions.cs`
- `TransactionIngressBackpressureInterceptor.cs`, `TransferConcurrencyGate.cs`
- Logs: `Transaction ingress backpressure`, `DiagnosticSummary=`

**Also:** [API reference](../reference/api.md), [gRPC services](../reference/grpc-services.md), [Load test reference runs](../load-testing/load-test-reference-runs.md).
