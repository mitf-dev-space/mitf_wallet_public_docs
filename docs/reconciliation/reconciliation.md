# Reconciliation Service

Daily reconciliation job: exports ledger entries for the previous day, matches them against bank statement entries, and stores run results and exceptions.

## Components

- **Ledger API**: gRPC `ExportEntries(ExportEntriesRequest)` returns all ledger entries in a date range (UTC). Response includes `idempotency_key`, amount, currency per entry. Reversal operations post new journal entries (e.g. key `reverse-{transactionId}`); included in the export like any other journal.
- **IBankStatementProvider**: Abstraction for bank statement data.
  - **MockBankStatementProvider**: Returns entries from config `Reconciliation:MockBank:Entries`. Use for development and tests.
  - **Real implementation**: Add a class implementing `IBankStatementProvider` (e.g. MT940/942 parser + bank API), register it in `Program.cs`.
- **Masarat.Reconciliation.Job**: Worker runs **catch-up** on each wake for recent calendar days without a Completed run (see `Reconciliation:MaxCatchUpDays`), then sleeps until the next `RunAtUtcHour` (default 02:00 UTC). Exports ledger, fetches bank entries, matches by reference and amount/currency, persists `ReconciliationRun` and `ReconciliationExceptions`.

## Matching rules

- **Reference**: Ledger `IdempotencyKey` matched to bank line `Reference`.
- **Match**: Same reference + same amount/currency → matched.
- **MissingInBank**: Ledger entry has no matching bank line.
- **MissingInLedger**: Bank line has no matching ledger entry.
- **AmountMismatch**: Same reference but amount or currency differs.

## Configuration

| Key | Description |
| --- | ----------- |
| `ConnectionStrings:Reconciliation` | PostgreSQL connection |
| `Reconciliation:LedgerGrpcAddress` | Ledger gRPC endpoint |
| `Reconciliation:RunAtUtcHour` | Hour in UTC for scheduled wake (default 2; clamped 0–23) |
| `Reconciliation:MaxCatchUpDays` | Days back to scan for incomplete dates (default 30) |
| `Reconciliation:AbandonedRunAgeHours` | Stale Running rows older than this are removed before retry (default 2; `0` = disabled) |
| `Reconciliation:BankAmountMatchTolerance` | Absolute tolerance for amount comparison (default 0) |
| `Observability:CollectorUrl` | Optional OTLP endpoint |

**Mock bank entries example:**

```json
"MockBank": {
  "IncludeUndatedEntries": false,
  "Entries": [
    { "Reference": "key-debit-123", "Amount": -100, "Currency": "LYD", "ValueDate": "2026-04-10T00:00:00Z" },
    { "Reference": "key-credit-123", "Amount": 100, "Currency": "LYD", "ValueDate": "2026-04-10T00:00:00Z" }
  ]
}
```

Lines with `ValueDate` are returned only when that UTC calendar day equals the reconciliation `runDate`. Undated lines are included only when `IncludeUndatedEntries=true` (default `true` for backward compatibility).

## Database

- **MasaratReconciliation**: Tables `ReconciliationRuns` (run summary) and `ReconciliationExceptions` (per-exception rows). Migrations run on startup.

## Adding a real bank implementation

1. Implement `IBankStatementProvider`: call your bank API or parse MT940/942 files; map each line to `BankStatementEntry(Reference, Amount, Currency, ValueDate)`.
2. Register in `Program.cs`: e.g. `builder.Services.AddSingleton<IBankStatementProvider, RealBankStatementProvider>();`
3. Ensure `Reference` in bank data matches what you store in the ledger (e.g. idempotency key or transaction ID).
