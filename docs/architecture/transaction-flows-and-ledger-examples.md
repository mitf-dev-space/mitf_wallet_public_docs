# Transaction Flows and Ledger Examples

Concrete examples of how each transaction type affects wallets and the ledger, including reversals (full and partial, with fee policy). Complements [Financial operations and reconciliation](../reconciliation/financial-operations-and-reconciliation.md).

## Conventions

- **Ledger**: Double-entry; each journal's legs sum to zero. **Debit = positive**, **credit = negative**.
- **Wallet balance**: Ledger balance of the wallet's liability account minus any locked amount.
- **Accounts**: Wallet A / B = customer wallet liabilities; Merchant Settlement = `Fees:MerchantSettlementAccountId`; Cash Settlement = `Fees:CashSettlementAccountId`; Fee Revenue = `Fees:FeeRevenueAccountId`.
- **Fees**: Added on top of principal; customer is debited `amount + fee`.
- **Reversal**: `ReverseTransaction(transaction_id, ..., amount?, fee_reversal_policy)`.
  - **Full reversal**: omit `amount` → entire principal reversed.
  - **Partial reversal**: provide `amount` → only that portion reversed; fee proportional when policy is `FULL`.
  - **fee_reversal_policy**: `FULL` (default) = fee reversed proportionally; `NONE` = fee not reversed.

All examples use **LYD** and rounded numbers.

---

## 2. P2P Transfer

### 2.1 Original (no reversal)

Wallet A sends **100 LYD** to Wallet B. Fee: **5 LYD**.

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Debit | +105 | −105 |
| Wallet B | Credit | −100 | +100 |
| Fee Revenue | Credit | −5 | +5 |

### 2.2 Full reversal, fee_reversal_policy = FULL

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Credit | −105 | +105 (full refund) |
| Wallet B | Debit | +100 | −100 |
| Fee Revenue | Debit | +5 | −5 |

Status → **Reversed**.

### 2.3 Partial reversal (40 LYD), fee_reversal_policy = FULL

Proportional fee: `5 × (40/100) = 2 LYD`.

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Credit | −42 | +42 (40 + 2 fee) |
| Wallet B | Debit | +40 | −40 |
| Fee Revenue | Debit | +2 | −2 |

Status remains **Completed**; further reversals possible up to remaining 60.

### 2.4 Partial reversal (40 LYD), fee_reversal_policy = NONE

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Credit | −40 | +40 (principal only) |
| Wallet B | Debit | +40 | −40 |
| Fee Revenue | — | 0 | No change |

---

## 3. Merchant Payment

### 3.1 Original (no reversal)

Wallet A pays **200 LYD**. Fee: **3 LYD**.

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Debit | +203 | −203 |
| Merchant Settlement | Credit | −200 | +200 |
| Fee Revenue | Credit | −3 | +3 |

### 3.2 Full reversal, fee_reversal_policy = FULL

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Credit | −203 | +203 |
| Merchant Settlement | Debit | +200 | −200 |
| Fee Revenue | Debit | +3 | −3 |

### 3.3 Full reversal, fee_reversal_policy = NONE

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Credit | −200 | +200 (principal only) |
| Merchant Settlement | Debit | +200 | −200 |
| Fee Revenue | — | 0 | No change |

---

## 4. Cash Withdrawal

### 4.1 Original (no reversal)

Wallet A withdraws **150 LYD**. Fee: **2 LYD**.

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Debit | +152 | −152 |
| Cash Settlement | Credit | −150 | +150 |
| Fee Revenue | Credit | −2 | +2 |

### 4.2 Full reversal, fee_reversal_policy = FULL

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Credit | −152 | +152 |
| Cash Settlement | Debit | +150 | −150 |
| Fee Revenue | Debit | +2 | −2 |

### 4.3 Partial reversal (50 LYD), fee_reversal_policy = FULL

Proportional fee: `2 × (50/150) ≈ 0.67 LYD`.

| Account | Leg | Amount (LYD) | Effect |
| ------- | --- | ------------ | ------ |
| Wallet A | Credit | −50.67 | +50.67 |
| Cash Settlement | Debit | +50 | −50 |
| Fee Revenue | Debit | +0.67 | −0.67 |

---

## 5. Flows that cannot be reversed via ReverseTransaction

| Flow | Ledger operation | Reversible via API |
| ---- | ---------------- | ------------------ |
| **Fund Wallet** | PostEntry: single credit to wallet | No |
| **Fund from Pool** | PostJournal: debit pool, credit wallet | No |

Any correction requires a separate operational flow (e.g. a manual debit entry).

---

## 6. Summary

| Type | Original movement | Reversal |
| ---- | ----------------- | -------- |
| **P2P** | Debit source (principal + fee); credit dest (principal); credit fee revenue | Credit source (reversed ± fee); debit dest; debit fee if FULL |
| **Merchant** | Debit wallet (principal + fee); credit merchant settlement; credit fee revenue | Credit wallet (reversed ± fee); debit settlement; debit fee if FULL |
| **Withdrawal** | Debit wallet (principal + fee); credit cash settlement; credit fee revenue | Credit wallet (reversed ± fee); debit cash settlement; debit fee if FULL |

## 7. Reversal parameters

| Parameter | Meaning |
| --------- | ------- |
| `amount` (optional) | Partial reversal amount. Omit for full principal reversal. |
| `fee_reversal_policy` | `FULL` (default): fee reversed proportionally. `NONE`: fee not reversed. |
| **Full reversal** | Omit `amount` → entire principal reversed; fee per policy → status **Reversed** |
| **Partial reversal** | Set `amount` → only that principal reversed; fee proportional if FULL → status stays **Completed** |

---

*See [Financial operations and reconciliation](../reconciliation/financial-operations-and-reconciliation.md) for reconciliation, exception handling, and full flow descriptions.*
