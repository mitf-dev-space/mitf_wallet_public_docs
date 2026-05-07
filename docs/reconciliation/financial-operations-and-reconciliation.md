# Financial Operations and Reconciliation

**Audience:** Business stakeholders, Finance, Operations, Audit.

Covers what each transaction type does and how it affects the ledger, how daily reconciliation runs and exceptions are identified, who resolves them, and where to find results.

The system uses a **double-entry ledger** as the single source of truth for balances. A **daily reconciliation job** compares ledger exports to bank statements and produces matched items and **exceptions** for follow-up.

---

## 1. Glossary

| Term | Definition |
| ---- | ---------- |
| **Ledger** | Central service (Ledger API) holding all financial entries. Balances computed from entries. |
| **Double-entry** | Every movement has debits and credits summing to zero for each journal. |
| **PostEntry** | Single-leg posting (one account). Used for wallet funding. |
| **PostJournal** | Multi-leg atomic posting (debit + credit ± fee). Used for transfers, merchant, withdrawal, reversal. |
| **Idempotency key** | Unique key per posting request. Duplicate key is rejected; makes retries safe and supports reconciliation by reference. |
| **Transaction (DB)** | Record in Transactions domain for money-moving operations. |
| **Reconciliation run** | One execution of the reconciliation job for D-1. Produces matched count and exception list. |
| **Exception** | A ledger line or bank line that could not be matched (MissingInBank, MissingInLedger, AmountMismatch). |

---

## 2. Ledger and Double-Entry Model

### 2.1 Ledger API and Accounts

- The **Ledger API** (port 5001) is the single source of truth via a **double-entry ledger**.
- Each **ledger account** has a **type** (Asset or Liability) and **currency** (e.g. LYD).
- **Balance** = sum of all `LedgerEntry` amounts for that account (computed from entries, not stored).

### 2.2 How Entries Are Created

- **PostEntry**: Single entry for **Fund Wallet** — one credit to the wallet's liability account.
- **PostJournal**: Atomic multi-leg posting (all legs sum to zero). Used for transfers, merchant, cash withdrawal, fund from pooled account, and reversals.

### 2.3 Idempotency

Every entry has an `IdempotencyKey`. For `PostJournal`, each leg has a distinct key (e.g. `{key}-debit`, `{key}-credit`, `{key}-fee`). Duplicate key → rejected; no double posting.

### 2.4 Double-Entry in this system

One **liability account per wallet**. No per-wallet asset account (common e-wallet pattern). The other side of each movement uses **shared** accounts: Cash Settlement, Merchant Settlement, Fee Revenue, Pool per product.

---

## 3. Transaction Types and Ledger Impact

### 3.1 P2P Transfer

| Step | Actor | Action |
| ---- | ----- | ------ |
| 1 | Client | `Transfer(fromWalletId, toWalletId, amount, currency, idempotencyKey)` |
| 2 | Transactions | Load wallets, check classification, get balance; FeeCalculator adds fee |
| 3 | Transactions | Reserve balance on source wallet; create Transaction (Pending) |
| 4 | Transactions | `PostJournal`: debit source liability, credit destination liability, optional fee revenue |
| 5 | Transactions | On success: Transaction → Completed; publish `TransferCompletedEvent`. On failure: Failed. |
| 6 | Transactions | Release reservation |

### 3.2 Fund Wallet (Top-Up)

| Step | Actor | Action |
| ---- | ----- | ------ |
| 1 | Client | `FundWallet(walletId, amount, currency, idempotencyKey[, linkedBankAccountId])` |
| 2 | Transactions | Validate wallet; allocate `transactionId` |
| 3 | Transactions | `PostEntry`: single credit to wallet liability |
| 4 | Transactions | Store idempotency; publish `WalletFundedEvent` |

No debit leg — current account debit is assumed external.

### 3.3 Merchant Payment

| Step | Actor | Action |
| ---- | ----- | ------ |
| 1 | Client | `ProcessMerchantPayment(walletId, amount, currency, idempotencyKey[, merchantReference])` |
| 2 | Transactions | Validate classification; FeeCalculator adds fee; check balance |
| 3 | Transactions | Reserve balance; create Transaction (Pending) |
| 4 | Transactions | `PostJournal`: debit wallet, credit Merchant settlement, optional fee revenue |
| 5 | Transactions | On success: Completed; publish `MerchantPaymentCompletedEvent`. On failure: Failed. |

### 3.4 Cash Withdrawal

| Step | Actor | Action |
| ---- | ----- | ------ |
| 1 | Client | `ProcessCashWithdrawal(walletId, amount, currency, idempotencyKey)` |
| 2 | Transactions | Validate classification; FeeCalculator adds fee; check balance |
| 3 | Transactions | Reserve balance; create Transaction (Pending) |
| 4 | Transactions | `PostJournal`: debit wallet, credit Cash settlement, optional fee revenue |
| 5 | Transactions | On success: Completed; publish `CashWithdrawalCompletedEvent`. On failure: Failed. |

### 3.5 Pooled Accounts

- **CreatePooledAccount**: Creates a pool and calls Ledger `CreateAccountsForWallet`(poolId, currency) → one liability account per pool.
- **FundWalletFromPooledAccount**: `PostJournal` (debit pool liability, credit wallet liability); publishes `WalletFundedEvent`.

### 3.6 Wallet Creation

`CreateWallet` (from Users, Customer Gateway, or LoadTest.Job) → Wallets API calls `CreateAccountsForWallet`(walletId, LYD) → one liability account per wallet.

### 3.7 Transaction Reversal

Reverses a **Completed** P2P, Merchant, or Withdrawal transaction by posting a balancing journal. `FundWallet` and `FundWalletFromPool` cannot be reversed via this API.

| Step | Actor | Action |
| ---- | ----- | ------ |
| 1 | Client | `ReverseTransaction(transaction_id, calling_bank_id[, reason, idempotency_key, amount, fee_reversal_policy])`. `x-bank-id` required. |
| 2 | Transactions | Load Transaction (must be Completed + same bank) |
| 3 | Transactions | Build reversal legs by type: P2P/Merchant/Withdrawal |
| 4 | Transactions | `PostJournal`: reversal id; idempotency key = `reverse-{originalTransactionId}` or client-provided |
| 5 | Transactions | On success: Transaction → **Reversed** (or stays Completed on partial) |

- **Partial reversal**: `amount` field = reverse only that portion. Fee proportional when policy is `FULL`.
- Status stays **Completed** on partial; further reversals possible.
- Idempotency: duplicate reverse calls with same key → rejected by Ledger.

---

## 4. Fees and Settlement Accounts

- **FeeCalculator** in Wallets applies rules by classification and transaction type. Fee added on top of principal.
- **Fee revenue** → `Fees:FeeRevenueAccountId`.
- **Merchant settlement** → `Fees:MerchantSettlementAccountId`.
- **Cash settlement** → `Fees:CashSettlementAccountId`.

These accounts must be created in Ledger (via seeding) and configured in Transactions API.

---

## 5. Transaction Lifecycle

Status: **Pending** → **Completed** or **Failed**; optionally **Reversed** (via `ReverseTransaction`).

### Recent upgrades

- Transaction detail now exposes ledger entries, reversal chain, and support metadata (reference, channel, counterparty, purpose, actor context).
- Wallet balance views distinguish available balance, locked balance, and ledger balance.
- Transaction history supports filtering by reference and amount range.
- Reversal flows support partial reversal and fee reversal policy, with linked reversal transactions visible in queries.
- Ledger balances backed by a derived snapshot for hot reads; entry ledger remains source of truth.

---

## 6. Reconciliation

### 6.1 Purpose

Compare **ledger entries** (our books) with **bank statement lines** (bank's view) for a given date to detect missing or mismatched items.

### 6.2 Schedule

- **Job:** `Masarat.Reconciliation.Job`, daily at configured UTC hour (default 02:00 UTC).
- **Run date:** Previous calendar day (D-1), full UTC day.
- **Idempotency:** One completed run per run date. Failed runs can be retried.

### 6.3 Steps

1. **Export:** Ledger gRPC `ExportEntries(FromDateUtc, ToDateUtc)` → entries with Id, AccountId, Amount, Currency, TransactionId, IdempotencyKey, CreatedAtUtc.
2. **Bank statement:** `IBankStatementProvider` — production = real bank feed (MT940/942 or bank API); development = `MockBankStatementProvider`.
3. **Matching:** `ReconciliationMatcher` matches ledger `IdempotencyKey` ↔ bank line `Reference` + amount/currency.
4. **Results:** Matched, MissingInBank, MissingInLedger, AmountMismatch.
5. **Persistence:** `ReconciliationRun` + `ReconciliationException` rows.

### 6.4 Bank Reference Alignment

For matching to succeed, the reference on the bank statement must align with what we store (e.g. idempotency key or transaction ID). This is integration-specific and must be agreed with the bank.

---

## 7. Reconciliation Exception Handling

### 7.1 Exception Types

| Exception | Meaning | Typical cause |
| --------- | ------- | ------------- |
| **MissingInBank** | Ledger entry has no matching bank line | Timing/cut-off; bank reporting error |
| **MissingInLedger** | Bank line has no matching ledger entry | Transaction not posted; wrong reference on bank line |
| **AmountMismatch** | Same reference but amount/currency differs | Incorrect amount posted; currency error; bank rounding |

### 7.2 Responsibilities

- **Daily review:** Designated person reviews each completed run and exception list.
- **MissingInBank:** Confirm if expected (next-day settlement). If persistent, escalate.
- **MissingInLedger:** Investigate why no ledger entry exists. If valid bank movement, coordinate with IT for corrective posting.
- **AmountMismatch:** Investigate root cause; correct or escalate.
- **Escalation:** Define path (Finance → Operations → IT / Bank) and SLA.

### 7.3 Retry

If the reconciliation job fails, it can be retried for the same run date. Only one Completed run per date is kept.

---

## 8. Reporting

- **`ReconciliationRun`:** RunDate, StartedAt, CompletedAt, Status, TotalExported, MatchedCount, ExceptionCount.
- **`ReconciliationException`:** RunId, ExceptionType, InternalReference, BankReference, ExpectedAmount, ActualAmount, Message.

---

## 9. Compliance and Audit

- **Audit trail:** All ledger entries are immutable with IdempotencyKey, TransactionId, and timestamps. Reversals create new entries with distinct keys.
- **Reconciliation as control:** Daily reconciliation and exception handling detect discrepancies between books and bank.
- **Retention:** Define retention policy for reconciliation runs and exceptions per regulatory requirements.

---

## 10. Summary Table

| Feature | Ledger operation | Transaction record | Event | In reconciliation |
| ------- | ---------------- | ------------------ | ----- | ----------------- |
| Wallet creation | CreateAccountsForWallet | No | WalletCreatedEvent | No |
| P2P Transfer | PostJournal (debit, credit [, fee]) | Yes | TransferCompletedEvent | Yes |
| Fund wallet | PostEntry (credit wallet) | No | WalletFundedEvent | Yes |
| Merchant payment | PostJournal (debit wallet, credit merchant [, fee]) | Yes | MerchantPaymentCompletedEvent | Yes |
| Cash withdrawal | PostJournal (debit wallet, credit cash [, fee]) | Yes | CashWithdrawalCompletedEvent | Yes |
| Create pooled account | CreateAccountsForWallet (pool) | No | — | No |
| Fund from pool | PostJournal (debit pool, credit wallet) | No | WalletFundedEvent | Yes |
| Transaction reversal | PostJournal (reverse principal ± fee) | Yes (→ Reversed) | — | Yes |

---

## 11. Important Details

- **Currency:** One currency per wallet (e.g. LYD); transfers and funding validate currency match.
- **Classification:** AllowP2P, AllowMerchant, AllowWithdrawal, PerTransactionMax, MaxBalance, CanSendInterBank, CanReceiveInterBank enforced before posting.
- **Balance check:** Available = Ledger balance − LockedBalance; reservation used during P2P, Merchant, and Cash flows.
- **Ledger journal:** All legs in one `PostJournal` written atomically; if any leg fails, whole journal rejected.
- **Reversal:** Principal full or partial. Fee reversed when `fee_reversal_policy=FULL` (proportional for partial). Fund Wallet and Fund from Pool have no Transaction record and cannot be reversed via `ReverseTransaction`.
