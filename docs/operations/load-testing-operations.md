# Load testing operations

How to run load tests, choose overlays, and interpret **async diagnostics**. For canonical results, see [Load test reference runs](../load-testing/load-test-reference-runs.md).

---

## Running overlays

1. **Base stack:** From the wallet application repository root, start services with `docker-compose.yml`.
2. **Load profile:** Apply a file under `compose/loadtest/` (e.g. `loadtest-250k-no-chaos.yml`) alongside the base compose.
3. **Automation:** Use `scripts/run-loadtests.ps1` for multi-scenario runs.
4. **Worker modes (`Masarat.LoadTest.Job`):**
   - **Direct gRPC:** `LoadTest__LoadTest__Enabled=true` + user/wallet/transaction addresses.
   - **Customer Gateway:** `LoadTest__CustomerGatewayLoadTest__Enabled=true`, `BaseUrl`, per-persona app IDs/keys, optional `Profile`.

Default compose enables **Customer Gateway mixed traffic** so management dashboards populate after startup. Set `LoadTest__CustomerGatewayLoadTest__Enabled=false` to keep the job idle.

---

## Async diagnostic maxima {#async-diagnostic-maxima}

The load job logs percentiles and **max** for async transfer phases:

- **max** is the largest single successful request's server-reported component — not the polling cap.
- Under chaos or heavy backlog, a few requests can sit in RabbitMQ for many minutes while **p50/p95** stay moderate, producing very large **max** values.
- The job emits a **warning** when any sample exceeds a threshold (e.g. 2 minutes). Inspect: queue depth, consumer count, Postgres pool size, ledger/transfer backpressure settings.

When comparing runs, prefer **p95/p99** and throughput for regression detection; use **max** for tail-risk postmortems.

---

## Postgres messaging hygiene (inbox / outbox / idle transactions)

After large runs (e.g. 1M transfers with many Transactions workers):

1. **Run snapshot** on `MasaratWallets`:
   ```
   docker exec -i masarat-db psql -U postgres -d MasaratWallets -f - < scripts/sql/masaratwallets-messaging-health-snapshot.sql
   ```
2. **`TransactionsInboxState` row count** should fall after load stops as MassTransit inbox cleanup runs. If it stays high for days, tune `Messaging:Tuning` on `Transactions.Api`.
3. **`idle in transaction`** sessions often correlate with long consume pipelines — EF outbox wraps the consumer in a DB transaction while the handler awaits gRPC (e.g. Ledger). A non-zero count under stress is not automatically a leak; large or long-lived `xact_age` values warrant investigation.
4. **Inbox cleanup deadlocks** on `TransactionsInboxState` may appear as retried warnings. Spikes or failures after retries are the real problem; occasional retries under peak load are a known contention pattern.

---

## Consistency checks

Sampled wallet balance sums and fee adjustments are logged as **PASS** or **WARN** depending on tolerance. Residuals under chaos are expected to be small relative to gross volume — see [load test reference runs](../load-testing/load-test-reference-runs.md) for interpretation.

---

- [Load test reference runs](../load-testing/load-test-reference-runs.md)
- [Stakeholder load test summary](../load-testing/stakeholder-load-test-summary.md)
- [Transfer backpressure client contract](../architecture/transfer-backpressure-client-contract.md)
