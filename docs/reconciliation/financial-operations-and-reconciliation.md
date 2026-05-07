# Financial Operations and Reconciliation

**Masarat (MITF) Wallet — Business & Finance Reference**

---

## Document Control


| Field              | Value                                                  |
| ------------------ | ------------------------------------------------------ |
| **Document title** | Financial Operations and Reconciliation                |
| **Version**        | 2.0                                                    |
| **Last updated**   | March 2026                                             |
| **Classification** | Internal — Business & Finance                          |
| **Owner**          | Development Team                                       |
| **Audience**       | Business stakeholders, Finance team, Operations, Audit |


---

## Executive Summary

This document describes how **transactions**, **ledger postings**, and **reconciliation** work in the Masarat (MITF) Wallet system. It is intended for the Business and Financial teams to understand:

- **What** each transaction type does and how it affects the ledger and customer balances.
- **How** daily reconciliation runs and how exceptions are identified and reported.
- **Who** is responsible for resolving reconciliation exceptions and when to escalate.
- **Where** to find reconciliation results and how they align with bank statements.

The system uses a **double-entry ledger** as the single source of truth for balances. All wallet movements (transfers, merchant payments, cash withdrawals, funding) post to the Ledger. A **daily reconciliation job** compares our ledger export to the bank’s statement for the previous day and produces a **reconciliation run** with matched items and **exceptions** (missing or mismatched items) for follow-up.

---

## 1. Glossary


| Term                   | Definition                                                                                                                           |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **Ledger**             | Central service (Ledger API) that holds all financial entries. Balances are computed from entries, not stored.                       |
| **Double-entry**       | Every movement is recorded with debits and credits so that total debits equal total credits for each journal.                        |
| **PostEntry**          | Single-leg posting (one account, one amount). Used for wallet funding from current account.                                          |
| **PostJournal**        | Multi-leg atomic posting (e.g. debit one account, credit another, optional fee). Used for transfers, merchant, withdrawal, reversal. |
| **Idempotency key**    | Unique key per posting request. Duplicate key is rejected; ensures no double posting and supports reconciliation by reference.       |
| **Transaction (DB)**   | Record in the Transactions domain for money-moving operations. P2P, Merchant, Withdrawal, Fund Wallet, and Fund from Pool flows all generate transaction identifiers and idempotency mappings, even though the storage model differs by flow. |
| **Reconciliation run** | One execution of the reconciliation job for a given date (D-1). Produces matched count and exception list.                           |
| **Exception**          | A ledger line or bank line that could not be matched (MissingInBank, MissingInLedger, AmountMismatch).                               |


---

## 2. Ledger and Double-Entry Model

### 2.1 Ledger API and Accounts

- The **Ledger API** (port 5001) is the **single source of truth** for balances via a **double-entry ledger**.
- Each **ledger account** has a **type** (Asset or Liability) and a **currency** (e.g. LYD).
- **Balance** for an account = sum of all **LedgerEntry** amounts for that account. Entries use **signed amounts** (e.g. negative = credit, positive = debit in the implemented convention).
- No balance is stored on the Ledger; it is always **computed from entries**.

### 2.2 How Entries Are Created

- **PostEntry**: Single entry (one account, one amount). Used for **Fund Wallet** (top-up from current account): one credit to the wallet’s liability account.
- **PostJournal**: Atomic **multi-leg** posting. All legs are written in one operation; the Ledger **validates that the sum of leg amounts is zero** (double-entry balance). Used for:
  - **Transfers** (debit source wallet, credit destination wallet, optional fee leg)
  - **Merchant payment** (debit wallet, credit merchant settlement, optional fee)
  - **Cash withdrawal** (debit wallet, credit cash settlement, optional fee)
  - **Fund from pooled account** (debit pool liability, credit wallet liability)
  - **Transaction reversal** (reverse principal and fee legs: credit source wallet, debit destination/settlement and fee revenue when applicable)

### 2.3 Idempotency at Ledger Level

- Every entry has an **IdempotencyKey**. For **PostJournal**, the key is **base + suffix** (e.g. `{clientKey}-debit`, `{clientKey}-credit`, `{clientKey}-fee`).
- Duplicate idempotency key → Ledger **rejects** the request (no duplicate entries). This makes retries safe and supports **reconciliation by reference**.

### 2.4 What “Double-Entry” Means Here (and How Others Do It)

- **In this system**, “double-entry” means: **every journal is balanced**. Each **PostJournal** has two or more legs whose amounts sum to zero (enforced by the Ledger). So every transfer is “debit source liability, credit destination liability”; every withdrawal is “debit wallet liability, credit cash settlement,” etc. Customer balance is held in **one ledger account per wallet — the liability account**. No per-wallet asset account is created or stored; this aligns with the common e-wallet pattern (one liability per wallet, shared asset accounts for settlement/fees/pools).
- **How other online systems typically do it**:
  - **E-wallets / fintech (Stripe, many wallet providers)**: One **liability** account per customer/wallet (what we owe the customer). The other side of each movement uses **shared** accounts: e.g. one Cash Settlement, one Merchant Settlement, one Fee Revenue, one Pool per product. So there is **no per-wallet “asset” account** — double-entry is “debit one liability, credit another” (transfer) or “debit wallet liability, credit shared asset” (withdrawal/merchant). This is the most common pattern.
  - **Funding from bank**: Often a **single credit** to the wallet liability; the matching debit stays in the bank’s core or external system. Our “Fund Wallet (PostEntry)” follows this.
  - **Full banking cores**: Some use both an asset and a liability per customer account for full balance-sheet reporting; that is heavier and usually not used for e-wallets.

---

## 3. Transaction Types and Ledger Impact

### 3.1 Wallet-to-Wallet Transfer (P2P)


| Step | Actor        | Action                                                                                                                                                                   |
| ---- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Client       | Calls **Transactions API** `Transfer(fromWalletId, toWalletId, amount, currency, idempotencyKey)`.                                                                       |
| 2    | Transactions | Loads wallets via Wallets API; checks classification (AllowP2P, limits, inter-bank rules); gets balance from Ledger; **FeeCalculator** (P2P) may add a fee.              |
| 3    | Transactions | Reserves balance on source wallet (locked), creates **Transaction** (Pending).                                                                                           |
| 4    | Transactions | Calls Ledger **PostJournal**: legs = (debit source liability, credit destination liability [, credit fee revenue account]).                                              |
| 5    | Transactions | On success: marks **Transaction** **Completed**, publishes **TransferCompletedEvent**, stores idempotency → transactionId. On failure: marks **Transaction** **Failed**. |
| 6    | Transactions | Releases reservation on source wallet.                                                                                                                                   |


- **Ledger legs**: Debit source wallet liability, credit destination wallet liability; optional third leg to fee revenue account.
- **Transaction record**: Stored in Transactions DB (id, type P2P, status, amount, currency, from/to wallet, created/completed at).

### 3.2 Fund Wallet (Top-Up from Current Account)


| Step | Actor        | Action                                                                                                               |
| ---- | ------------ | -------------------------------------------------------------------------------------------------------------------- |
| 1    | Client       | Calls **Transactions API** `FundWallet(walletId, amount, currency, idempotencyKey[, linkedBankAccountId])`.          |
| 2    | Transactions | Validates wallet (active, same bank, currency) and allocates a **transactionId** for idempotent replay and audit correlation. |
| 3    | Transactions | Calls Ledger **PostEntry**: single **credit** to wallet’s liability account (amount, transactionId, idempotencyKey). |
| 4    | Transactions | On success: stores idempotency → transactionId and publishes **WalletFundedEvent**. |


- **Ledger**: One entry (credit to wallet liability). No debit leg in our system; the “current account” is assumed external.
- **Transaction record**: The flow still generates and stores a `transactionId` through the funding idempotency path so retries and downstream events can be correlated consistently.

### 3.3 Merchant Payment


| Step | Actor        | Action                                                                                                                                                                          |
| ---- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Client       | Calls **Transactions API** `ProcessMerchantPayment(walletId, amount, currency, idempotencyKey[, merchantReference])`.                                                           |
| 2    | Transactions | Validates wallet and classification (AllowMerchant, per-transaction limit); **FeeCalculator** (Merchant) may add a fee; checks balance.                                         |
| 3    | Transactions | Reserves balance (amount + fee), creates **Transaction** (Pending).                                                                                                             |
| 4    | Transactions | Calls Ledger **PostJournal**: debit wallet liability (amount + fee), credit **Merchant settlement account**, optional credit **Fee revenue account**.                           |
| 5    | Transactions | On success: marks **Transaction** **Completed**, publishes **MerchantPaymentCompletedEvent**, stores idempotency. On failure: **Transaction** **Failed**. Releases reservation. |


- **Ledger legs**: Debit wallet, credit merchant settlement (configurable `Fees:MerchantSettlementAccountId`), optional fee revenue.
- **Transaction record**: Stored (type Merchant, status, amount, fee, etc.).

### 3.4 Cash Withdrawal


| Step | Actor        | Action                                                                                                                                                                         |
| ---- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Client       | Calls **Transactions API** `ProcessCashWithdrawal(walletId, amount, currency, idempotencyKey)`.                                                                                |
| 2    | Transactions | Validates wallet and classification (AllowWithdrawal, limits); **FeeCalculator** (Withdrawal) may add a fee; checks balance.                                                   |
| 3    | Transactions | Reserves balance (amount + fee), creates **Transaction** (Pending).                                                                                                            |
| 4    | Transactions | Calls Ledger **PostJournal**: debit wallet liability (amount + fee), credit **Cash settlement account**, optional credit **Fee revenue account**.                              |
| 5    | Transactions | On success: marks **Transaction** **Completed**, publishes **CashWithdrawalCompletedEvent**, stores idempotency. On failure: **Transaction** **Failed**. Releases reservation. |


- **Ledger legs**: Debit wallet, credit cash settlement (configurable `Fees:CashSettlementAccountId`), optional fee revenue.
- **Transaction record**: Stored (type Withdrawal, status, amount, fee, etc.).

### 3.5 Pooled Accounts (A3mal / Bank Pool)

- **CreatePooledAccount**: Creates a **pool** (Transactions/Wallets domain) and calls Ledger **CreateAccountsForWallet**(poolId, currency). The Ledger creates **one liability account** per pool; it represents the pool’s obligation and is used for funding.
- **FundWalletFromPooledAccount**: Transactions API allocates a **transactionId**, calls Ledger **PostJournal** (**debit** pool liability, **credit** wallet liability), stores idempotency, and publishes **WalletFundedEvent**.

### 3.6 Wallet Creation and Ledger

- **CreateWallet** (from Users onboarding, Customer Gateway flows, or **Masarat.LoadTest.Job**): Wallets API creates the wallet, then calls Ledger **CreateAccountsForWallet**(walletId, LYD). Ledger creates **one liability account** per wallet. Wallet balance shown to the user is that **liability account** balance (from Ledger) minus any **locked** amount.

### 3.7 Transaction Reversal

- **ReverseTransaction**: Reverses a **completed** transaction (P2P, Merchant, or Withdrawal) by posting a **balancing journal** that undoes the principal movement. Only transactions that have a **Transaction** record in the Transactions DB can be reversed; **Fund Wallet** and **Fund Wallet from Pool** do not create a transaction record and have **no reversal** via this API.


| Step | Actor        | Action                                                                                                                                                                                                                                |
| ---- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Client       | Calls **Transactions API** (gRPC) `ReverseTransaction(transaction_id, calling_bank_id[, reason, idempotency_key, amount, fee_reversal_policy])`. **x-bank-id** (calling bank) is required. Optional **amount** = partial reversal amount; when omitted, full principal is reversed. **fee_reversal_policy**: `FULL` (default, fee reversed proportionally) or `NONE`. |
| 2    | Transactions | Loads **Transaction** by id; ensures status is **Completed**; ensures transaction belongs to calling bank (FromWallet or ToWallet in same bank).                                                                                      |
| 3    | Transactions | Builds **reversal legs** by transaction type: **P2P**: credit source wallet (reversed principal + fee when FULL), debit destination wallet (reversed principal), optional debit fee revenue (fee when FULL). **Merchant**: credit wallet (reversed principal + fee when FULL), debit merchant settlement (reversed principal), optional debit fee revenue. **Withdrawal**: credit wallet (reversed principal + fee when FULL), debit cash settlement (reversed principal), optional debit fee revenue. |
| 4    | Transactions | Calls Ledger **PostJournal**: new **transactionId** (reversal), idempotency key = `reverse-{originalTransactionId}` or client-provided; legs reverse **principal (full or partial) and fee (per fee_reversal_policy)** so the journal balances.                                                                 |
| 5    | Transactions | On success: marks **Transaction** **Reversed** (with optional reason), or remains **Completed** if partial reversal. On failure: returns error; transaction status unchanged.                                                                                                  |


- **Ledger legs**: Reversal posts **principal (full or partial)** and optionally **fee** (when `fee_reversal_policy` is FULL, fee is reversed proportionally): the customer’s wallet is credited; settlement and fee revenue (when configured) are debited so the journal balances.
- **Partial reversal**: Request can include optional **amount** to reverse only part of the principal; fee reversal is proportional when policy is FULL.
- **Transaction record**: Status changes from **Completed** to **Reversed** (or remains Completed if only a partial reversal was applied and further reversals are possible). **ErrorMessage** can store the reason. The original transaction id is unchanged; the reversal creates new ledger entries under a new transaction id for audit.
- **Idempotency**: Reversal uses a distinct idempotency key (e.g. `reverse-{transactionId}` or client-supplied). Duplicate reverse calls with the same key are rejected by the Ledger.
- **Reconciliation**: Reversal entries are exported by **ExportEntries** like any other journal; they can be matched to bank lines if the bank also reports the reversal with a matching reference.

---

## 4. Fees and Settlement Accounts

- **FeeCalculator** (in Wallets) applies rules by **classification** and **transaction type** (P2P, Merchant, Withdrawal). Fee is added on top of the principal amount; the customer is debited **amount + fee** where applicable.
- **Fee revenue** is posted to a configurable ledger account (`Fees:FeeRevenueAccountId`).
- **Merchant** and **Cash** flows credit configurable settlement accounts (`Fees:MerchantSettlementAccountId`, `Fees:CashSettlementAccountId`). These are ledger accounts (typically system/operational accounts) used for reconciliation with bank or internal settlement.
- Fee revenue and settlement accounts must be created in the Ledger (e.g. via seeding) and configured in Transactions API so that fee and settlement legs post to the correct accounts.

---

## 5. Transaction Lifecycle (Transactions API)

- **Transaction** entity (when used) has: Id, Type (P2P, Merchant, Withdrawal), **Status** (Pending → Completed or Failed; optionally Reversed), Amount, Fee, Currency, FromWalletId/ToWalletId, CreatedAt, CompletedAt, ErrorMessage.
- **Pending**: Created before Ledger call.
- **Completed**: Ledger posted successfully; event published.
- **Failed**: Validation or Ledger error; optional ErrorMessage.
- **Reversed**: Was completed but later reversed via **ReverseTransaction**. Ledger gets a balancing journal (principal and fee); transaction status set to Reversed with optional reason.

### 5.1 Recent product and operational upgrades

- Transaction detail now exposes ledger entries, reversal chain, and support metadata such as reference, channel, counterparty, purpose, and actor context.
- Wallet balance views now distinguish available balance, locked balance, and ledger balance.
- Transaction history now supports stronger filtering by reference and amount range alongside the existing type, status, and date filters.
- Reversal flows now support partial reversal and fee reversal policy, with linked reversal transactions visible in queries.
- Ledger balances are additionally backed by a derived snapshot for hot reads, while the entry ledger remains the source of truth.

---

## 6. Reconciliation

### 6.1 Purpose

Reconciliation compares **ledger entries** (our books) with **bank statement lines** (bank’s view) for a given date to detect missing or mismatched items. It is the primary control for ensuring that what we have recorded matches what the bank has recorded.

### 6.2 Schedule and Run Date

- **Job**: **Masarat.Reconciliation.Job** (part of the batch/scheduled jobs).
- **Schedule**: Runs **daily** at a configured UTC hour (e.g. 02:00 UTC).
- **Run date**: Always for the **previous calendar day** (D-1). The export window is that full UTC day (00:00 UTC to 00:00 UTC next day).
- **Idempotency**: One completed run per run date; if a run for that date already exists with status **Completed**, the job skips. Failed runs can be retried (no duplicate completed run for same date).

### 6.3 Reconciliation Steps

1. **Export from Ledger**
  The job calls Ledger gRPC **ExportEntries(FromDateUtc, ToDateUtc)** for the run date. Response: list of entries with **Id**, **AccountId**, **Amount**, **Currency**, **TransactionId**, **IdempotencyKey**, **CreatedAtUtc**.
2. **Bank statement**
  The job uses **IBankStatementProvider** to get bank entries for the same date. In production this would be a real bank feed (e.g. MT940/942 or bank API); in development **MockBankStatementProvider** returns configured entries. Each bank line has **Reference**, **Amount**, **Currency**, **ValueDate**.
3. **Matching**
  **ReconciliationMatcher** matches:
  - **Reference** = Ledger entry **IdempotencyKey** ↔ Bank line **Reference**.
  - **Amount** and **Currency** must match.
4. **Results**
  - **Matched**: Same reference and same amount/currency → counted as matched.
  - **MissingInBank**: Ledger entry has no bank line with that reference.
  - **MissingInLedger**: Bank line has no ledger entry with that reference.
  - **AmountMismatch**: Same reference but amount or currency differs.
5. **Persistence**
  - **ReconciliationRun**: RunDate, StartedAt, CompletedAt, Status (Running → Completed or Failed), TotalExported, MatchedCount, ExceptionCount, ErrorMessage.
  - **ReconciliationException** rows: RunId, ExceptionType (MissingInBank, MissingInLedger, AmountMismatch), InternalReference (ledger idempotency key), BankReference, ExpectedAmount, ActualAmount, Message.

### 6.4 Link to Transaction Flows

- Every **ledger entry** created by our flows has an **IdempotencyKey**. For **PostJournal**, each leg has a distinct key (base + suffix, e.g. `idemKey-debit`, `idemKey-credit`, `idemKey-fee`).
- For reconciliation to work with the bank, **references** sent in payments (e.g. transaction ID or the same idempotency key) must be what the bank returns on the statement. Then the matcher can pair ledger entries to bank lines.
- **Fund Wallet** (PostEntry) and **Fund Wallet from Pool** (PostJournal) also produce entries with idempotency keys; they are included in **ExportEntries** and thus in reconciliation.

### 6.5 Bank Reference Alignment

- For matching to succeed, the **reference** on the bank statement must align with what we store (e.g. idempotency key or transaction ID). This is **integration-specific** and must be agreed with the bank or payment provider.

---

## 7. Reconciliation Exception Handling

### 7.1 Exception Types


| Exception type      | Meaning                                           | Typical cause                                                                 |
| ------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------- |
| **MissingInBank**   | We have a ledger entry but no matching bank line. | Bank has not yet reported the item; timing/ cut-off; or bank reporting error. |
| **MissingInLedger** | We have a bank line but no matching ledger entry. | We did not post the transaction; or wrong reference on bank line.             |
| **AmountMismatch**  | Same reference but amount or currency differs.    | Incorrect amount posted; currency error; or bank rounding/format difference.  |


### 7.2 Responsibilities (Business / Finance)

- **Daily review**: Designated person(s) should review each completed reconciliation run and the exception list.
- **MissingInBank**: Confirm whether the item is expected (e.g. next-day settlement). If persistent, escalate to Operations/IT and/or bank.
- **MissingInLedger**: Investigate why no ledger entry exists. If it is a valid bank movement, coordinate with IT for corrective posting and process improvement.
- **AmountMismatch**: Investigate root cause (our posting vs bank data). Correct ledger if we are wrong; escalate to bank if they are wrong.
- **Escalation**: Define and document escalation path (e.g. Finance → Operations → IT / Bank) and SLA for resolution (e.g. within 2 business days for exceptions above a threshold).

### 7.3 Retry and Re-run

- If the reconciliation job **fails** (e.g. Ledger or bank feed unavailable), it can be **retried** for the same run date. Only one **Completed** run per run date is kept; retrying does not create a duplicate completed run.

---

## 8. Reporting and Outputs

- **ReconciliationRun**: For each run date, the system stores RunDate, StartedAt, CompletedAt, Status, TotalExported, MatchedCount, ExceptionCount, ErrorMessage. This can be used for daily dashboards and audit.
- **ReconciliationException**: Each exception stores RunId, ExceptionType, InternalReference, BankReference, ExpectedAmount, ActualAmount, Message. These records support exception reports and follow-up by Finance/Operations.
- **Export**: Reconciliation data is stored in the Reconciliation Job database (e.g. **ReconciliationRuns**, **ReconciliationExceptions** tables). Reports and exports can be built on top of this data for the Business and Financial teams.

---

## 9. Compliance and Audit

- **Audit trail**: All ledger entries are immutable and carry IdempotencyKey, TransactionId, and timestamps. Reversals create new entries with a distinct idempotency key and transaction id.
- **Reconciliation as control**: Daily reconciliation and exception handling are key controls for detecting discrepancies between our books and the bank.
- **Retention**: Define retention policy for reconciliation runs and exception records in line with regulatory and internal audit requirements.

---

## 10. Summary Table


| Feature                           | Ledger operation                                    | Transaction record (Transactions DB) | Event                         | In reconciliation export                  |
| --------------------------------- | --------------------------------------------------- | ------------------------------------ | ----------------------------- | ----------------------------------------- |
| **Wallet creation**               | CreateAccountsForWallet (one liability account)     | No                                   | WalletCreatedEvent            | No (no entries yet)                       |
| **P2P Transfer**                  | PostJournal (debit, credit [, fee])                 | Yes (P2P, Pending/Completed/Failed)  | TransferCompletedEvent        | Yes (each leg by idempotency key)         |
| **Fund wallet (current account)** | PostEntry (credit wallet)                           | No                                   | WalletFundedEvent             | Yes                                       |
| **Merchant payment**              | PostJournal (debit wallet, credit merchant [, fee]) | Yes (Merchant)                       | MerchantPaymentCompletedEvent | Yes                                       |
| **Cash withdrawal**               | PostJournal (debit wallet, credit cash [, fee])     | Yes (Withdrawal)                     | CashWithdrawalCompletedEvent  | Yes                                       |
| **Create pooled account**         | CreateAccountsForWallet (one liability for pool)   | No                                   | —                             | No                                        |
| **Fund from pool**                | PostJournal (debit pool, credit wallet)             | No                                   | WalletFundedEvent             | Yes                                       |
| **Transaction reversal**          | PostJournal (reverse principal ± fee per policy)    | Yes (status → Reversed)              | —                             | Yes (reversal entries by idempotency key) |


---

## 11. Important Details (Reference)

- **Currency**: One currency per wallet (e.g. LYD); transfers and funding validate currency match.
- **Classification**: AllowP2P, AllowMerchant, AllowWithdrawal, PerTransactionMax, MaxBalance, CanSendInterBank, CanReceiveInterBank are enforced before posting.
- **Balance check**: Available = Ledger balance − LockedBalance; reservation (TryReserveBalance) is used during P2P, Merchant, and Cash flows to avoid over-debit.
- **Ledger journal**: All legs in one **PostJournal** are written atomically; if any leg fails (e.g. duplicate idempotency), the whole journal is rejected.
- **Reversal**: **Principal** can be reversed in full or in part (optional `amount`). **Fee** is reversed when `fee_reversal_policy` is FULL (proportionally for partial reversals), or left as-is when NONE. **Fund Wallet** and **Fund from Pool** have no Transaction record and cannot be reversed via ReverseTransaction; any correction would require a separate operational flow (e.g. debit entry).

---

## Appendix A — Document History


| Version | Date       | Author | Changes                                                                                                                                          |
| ------- | ---------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1.0     | —          | —      | Initial technical document                                                                                                                       |
| 2.0     | May 2026 | —      | Restructured for Business & Finance: document control, executive summary, glossary, roles, exception handling, reporting, compliance, formatting |


---

*End of document.*