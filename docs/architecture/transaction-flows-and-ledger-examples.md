# Transaction Flows and Ledger Examples

**Masarat (MITF) Wallet — How transactions move money and affect wallets and the ledger**

This document shows **concrete examples** of how each transaction type (Transfer, Merchant Payment, Cash Withdrawal) affects **wallets** and the **ledger**, and how **reversals** (full and partial, with fee policy) undo or partly undo those movements. It complements [Financial operations and reconciliation](../reconciliation/financial-operations-and-reconciliation.md).

---

## Document Control

| Field              | Value                                   |
| ------------------ | --------------------------------------- |
| **Document title** | Transaction Flows and Ledger Examples   |
| **Version**        | 1.0                                     |
| **Last updated**   | May 2026                                 |
| **Audience**       | Developers, Operations, Support, Finance |

---

## 1. Conventions Used in This Document

- **Ledger**: Double-entry; each journal’s legs sum to zero. Amounts below use **signed** values: **debit = positive**, **credit = negative** (or the reverse depending on implementation; the important point is that debits and credits balance).
- **Wallet balance**: Shown to the user = Ledger balance of the wallet’s **liability account** minus any **locked** amount.
- **Accounts**:
  - **Wallet A / Wallet B**: Customer wallet liability accounts.
  - **Merchant Settlement**: System account for merchant payouts (`Fees:MerchantSettlementAccountId`).
  - **Cash Settlement**: System account for cash withdrawals (`Fees:CashSettlementAccountId`).
  - **Fee Revenue**: System account for fees (`Fees:FeeRevenueAccountId`).
- **Fees**: Added **on top of** principal by FeeCalculator; customer is debited **principal + fee** where applicable.
- **Reversal**: `ReverseTransaction(transaction_id, ..., amount?, fee_reversal_policy)`.
  - **Full reversal**: omit `amount` → entire principal (and optionally fee) reversed.
  - **Partial reversal**: provide `amount` → only that portion of principal reversed; fee reversal is **proportional** when policy is `FULL`.
  - **fee_reversal_policy**: `FULL` (default) = fee reversed proportionally; `NONE` = fee not reversed.

All examples use **LYD** and rounded numbers for clarity.

---

## 2. Wallet-to-Wallet Transfer (P2P)

### 2.1 Original transfer (no reversal)

**Setup**: Wallet A (source) sends **100 LYD** to Wallet B (destination). FeeCalculator applies a **5 LYD** P2P fee.

| Actor        | Action |
| ------------ | ------ |
| Client       | Calls `Transfer(WalletA, WalletB, 100, LYD, idem-t1)`. |
| Transactions | FeeCalculator adds fee 5; reserves 105 on Wallet A; creates Transaction (Pending). |
| Ledger       | **PostJournal** with 3 legs. |

**Ledger legs (PostJournal):**

| Account           | Leg   | Amount (LYD) | Effect on balance        |
| ----------------- | ----- | ------------ | ------------------------ |
| Wallet A (source) | Debit | +105         | Wallet A balance −105    |
| Wallet B (dest)   | Credit| −100         | Wallet B balance +100    |
| Fee Revenue       | Credit| −5           | Fee revenue +5           |

**Wallet view after transfer:**

- **Wallet A**: −105 (sent 100 + 5 fee).
- **Wallet B**: +100 (received 100).
- **Transaction**: Type P2P, Status **Completed**, Amount 100, Fee 5.

---

### 2.2 Reversal: full principal, fee_reversal_policy = FULL

**Action**: `ReverseTransaction(t1, reason="Customer request")` — no `amount`, so full reversal; default `FULL` fee policy.

**Reversal ledger legs:**

| Account           | Leg    | Amount (LYD) | Effect                          |
| ----------------- | ------ | ------------ | ------------------------------- |
| Wallet A (source) | Credit | −105         | Wallet A balance +105 (refund)   |
| Wallet B (dest)   | Debit  | +100         | Wallet B balance −100           |
| Fee Revenue       | Debit  | +5           | Fee revenue −5 (fee refunded)    |

**Result**: Wallet A gets 105 back (principal + fee). Wallet B loses 100. Transaction status → **Reversed**.

---

### 2.3 Reversal: partial principal (40 LYD), fee_reversal_policy = FULL

**Action**: `ReverseTransaction(t1, amount=40, fee_reversal_policy=FULL)`.

Proportional fee for reversed amount: `5 × (40/100) = 2 LYD`.

**Reversal ledger legs:**

| Account           | Leg    | Amount (LYD) | Effect                          |
| ----------------- | ------ | ------------ | ------------------------------- |
| Wallet A (source) | Credit | −42          | Wallet A +42 (40 principal + 2 fee) |
| Wallet B (dest)   | Debit  | +40          | Wallet B −40                    |
| Fee Revenue       | Debit  | +2           | Fee revenue −2                  |

**Result**: Customer (Wallet A) is refunded 40 + proportional fee 2. Transaction remains **Completed** (partial reversal; further reversals possible up to remaining 60).

---

### 2.4 Reversal: partial principal (40 LYD), fee_reversal_policy = NONE

**Action**: `ReverseTransaction(t1, amount=40, fee_reversal_policy=NONE)`.

**Reversal ledger legs:**

| Account           | Leg    | Amount (LYD) | Effect                          |
| ----------------- | ------ | ------------ | ------------------------------- |
| Wallet A (source) | Credit | −40          | Wallet A +40 (principal only)    |
| Wallet B (dest)   | Debit  | +40          | Wallet B −40                    |
| Fee Revenue       | —      | 0            | No change (fee not reversed)     |

**Result**: Customer gets 40 back; the original 5 LYD fee is not refunded. Transaction remains **Completed**.

---

## 3. Merchant Payment

### 3.1 Original merchant payment (no reversal)

**Setup**: Wallet A pays a merchant **200 LYD**. FeeCalculator applies **3 LYD** merchant fee.

| Actor        | Action |
| ------------ | ------ |
| Client       | Calls `ProcessMerchantPayment(WalletA, 200, LYD, idem-m1)`. |
| Transactions | FeeCalculator adds fee 3; reserves 203; creates Transaction (Pending). |
| Ledger       | **PostJournal**: debit wallet (amount + fee), credit merchant settlement, credit fee revenue. |

**Ledger legs (PostJournal):**

| Account             | Leg   | Amount (LYD) | Effect                    |
| ------------------- | ----- | ------------ | ------------------------- |
| Wallet A            | Debit | +203         | Wallet A balance −203     |
| Merchant Settlement | Credit| −200         | Merchant settlement +200  |
| Fee Revenue         | Credit| −3           | Fee revenue +3            |

**Wallet view after payment:**

- **Wallet A**: −203 (200 to merchant + 3 fee).
- **Merchant Settlement**: +200 (to be paid out to merchant).
- **Transaction**: Type Merchant, Status **Completed**, Amount 200, Fee 3.

---

### 3.2 Reversal: full principal, fee_reversal_policy = FULL

**Action**: `ReverseTransaction(m1, reason="Refund requested", fee_reversal_policy=FULL)`.

**Reversal ledger legs:**

| Account             | Leg    | Amount (LYD) | Effect                         |
| ------------------- | ------ | ------------ | ------------------------------ |
| Wallet A            | Credit | −203         | Wallet A +203 (full refund)     |
| Merchant Settlement | Debit  | +200         | Merchant settlement −200       |
| Fee Revenue         | Debit  | +3           | Fee revenue −3                 |

**Result**: Customer gets 203 back; merchant settlement and fee revenue are reduced. Transaction status → **Reversed**.

---

### 3.3 Reversal: full principal, fee_reversal_policy = NONE

**Action**: `ReverseTransaction(m1, fee_reversal_policy=NONE)`.

**Reversal ledger legs:**

| Account             | Leg    | Amount (LYD) | Effect                         |
| ------------------- | ------ | ------------ | ------------------------------ |
| Wallet A            | Credit | −200         | Wallet A +200 (principal only) |
| Merchant Settlement | Debit  | +200         | Merchant settlement −200       |
| Fee Revenue         | —      | 0            | No change                      |

**Result**: Customer gets 200 back (no fee refund). Merchant settlement reduced by 200. Transaction status → **Reversed**.

---

## 4. Cash Withdrawal

### 4.1 Original cash withdrawal (no reversal)

**Setup**: Wallet A withdraws **150 LYD** cash. FeeCalculator applies **2 LYD** withdrawal fee.

| Actor        | Action |
| ------------ | ------ |
| Client       | Calls `ProcessCashWithdrawal(WalletA, 150, LYD, idem-w1)`. |
| Transactions | FeeCalculator adds fee 2; reserves 152; creates Transaction (Pending). |
| Ledger       | **PostJournal**: debit wallet (amount + fee), credit cash settlement, credit fee revenue. |

**Ledger legs (PostJournal):**

| Account         | Leg   | Amount (LYD) | Effect                   |
| --------------- | ----- | ------------ | ------------------------ |
| Wallet A        | Debit | +152         | Wallet A balance −152    |
| Cash Settlement | Credit| −150         | Cash settlement +150    |
| Fee Revenue     | Credit| −2           | Fee revenue +2           |

**Wallet view after withdrawal:**

- **Wallet A**: −152 (150 cash + 2 fee).
- **Cash Settlement**: +150 (cash paid out).
- **Transaction**: Type Withdrawal, Status **Completed**, Amount 150, Fee 2.

---

### 4.2 Reversal: full principal, fee_reversal_policy = FULL

**Action**: `ReverseTransaction(w1, reason="ATM error – cash not dispensed", fee_reversal_policy=FULL)`.

**Reversal ledger legs:**

| Account         | Leg    | Amount (LYD) | Effect                        |
| --------------- | ------ | ------------ | ----------------------------- |
| Wallet A        | Credit | −152         | Wallet A +152 (full refund)    |
| Cash Settlement | Debit  | +150         | Cash settlement −150          |
| Fee Revenue     | Debit  | +2           | Fee revenue −2                |

**Result**: Customer gets 152 back; cash settlement and fee revenue reduced. Transaction status → **Reversed**.

---

### 4.3 Reversal: partial principal (50 LYD), fee_reversal_policy = FULL

**Action**: `ReverseTransaction(w1, amount=50, fee_reversal_policy=FULL)`.

Proportional fee: `2 × (50/150) ≈ 0.67 LYD` (system may round per configuration).

**Reversal ledger legs (assuming 0.67):**

| Account         | Leg    | Amount (LYD) | Effect                             |
| --------------- | ------ | ------------ | ---------------------------------- |
| Wallet A        | Credit | −50.67       | Wallet A +50.67 (50 + proportional fee) |
| Cash Settlement | Debit  | +50          | Cash settlement −50                |
| Fee Revenue     | Debit  | +0.67        | Fee revenue −0.67                  |

**Result**: Customer refunded 50 principal + proportional fee. Transaction remains **Completed** (partial; remaining reversible amount 100).

---

## 5. Flows That Do Not Support ReverseTransaction

These flows do **not** create a **Transaction** record in the Transactions DB, so they **cannot** be reversed via `ReverseTransaction`:

| Flow                    | Ledger operation                          | Reversal via API |
| ----------------------- | ----------------------------------------- | ----------------- |
| **Fund Wallet**         | PostEntry: single credit to wallet        | No                |
| **Fund from Pool**      | PostJournal: debit pool, credit wallet    | No                |

Any correction for these would require a separate operational flow (e.g. a debit entry or manual adjustment).

---

## 6. Summary: Movement by Transaction Type

| Type        | Original movement (wallet / ledger)                                                                 | Reversal movement (when applied)                                                                 |
| ----------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **P2P**     | Debit source wallet (principal + fee); credit destination (principal); credit fee revenue (fee).    | Credit source (reversed principal ± fee); debit destination (reversed principal); debit fee (if FULL). |
| **Merchant**| Debit wallet (principal + fee); credit merchant settlement (principal); credit fee revenue (fee).    | Credit wallet (reversed principal ± fee); debit merchant settlement; debit fee revenue (if FULL). |
| **Withdrawal** | Debit wallet (principal + fee); credit cash settlement (principal); credit fee revenue (fee).     | Credit wallet (reversed principal ± fee); debit cash settlement; debit fee revenue (if FULL).   |

---

## 7. Quick Reference: Reversal Parameters

| Parameter             | Meaning |
| --------------------- | ------- |
| `amount` (optional)   | Partial reversal amount. Omit for **full** principal reversal. |
| `fee_reversal_policy` | `FULL` (default): fee reversed proportionally. `NONE`: fee not reversed. |
| **Full reversal**     | Omit `amount` → entire principal reversed; fee per policy. Status → **Reversed**. |
| **Partial reversal**  | Set `amount` → only that principal reversed; fee proportional if FULL. Status stays **Completed**. |

---

*End of document. See [Financial operations and reconciliation](../reconciliation/financial-operations-and-reconciliation.md) for reconciliation, exception handling, and full flow descriptions.*
