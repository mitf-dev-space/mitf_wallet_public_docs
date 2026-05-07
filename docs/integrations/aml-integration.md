# AML integration (Masarat Wallet → FlowGuard) {: .wallet-lead }

After a wallet money movement commits, domain events are consumed by `Masarat.AmlBridge`, which maps them to FlowGuard's `TransactionQueueMessage` shape and publishes to RabbitMQ — **without blocking** the transaction or ledger path.

## End-to-end message flow

```mermaid
sequenceDiagram
  autonumber
  participant TX as Transactions / Wallets
  participant OB as Outbox + broker
  participant BR as AmlBridge worker
  participant FG as FlowGuard (RabbitMQ consumer)

  TX->>TX: Persist business outcome
  TX->>OB: Outbox publish intent (same DB txn)
  OB->>BR: Domain event delivered
  BR->>BR: Resolve BankId → BankCode
  BR->>FG: Publish transaction.{BankCode} on aml.transactions
```

| Piece | Role |
| ----- | ---- |
| **Domain events** (`Masarat.MessagingContracts`) | `TransferCompletedEvent`, `WalletFundedEvent`, `MerchantPaymentCompletedEvent`, `CashWithdrawalCompletedEvent`, `TransactionReversedEvent` |
| **`Masarat.AmlBridge`** | Worker: MassTransit consumers → map to `TransactionQueueMessage` → publish to topic exchange `aml.transactions` with routing key `transaction.{BankCode}` |

## Contract (FlowGuard)

Payload shape and exchange naming are documented in the AML repository:

`docs/integrations/masarat-wallet-flowguard-integration.md` (in the **AMLSystem** / FlowGuard repo).

## Configuration

Bridge settings under **`AmlIntegration`**:

- **`Enabled`**: when `false`, consumers no-op.
- **`FlowGuardExchangeName`**: default `aml.transactions`.
- **`FlowGuardRoutingKeyTemplate`**: default `transaction.{BankCode}`.
- **`BankCodes`**: object mapping bank Guid strings to FlowGuard bank codes (e.g. `JUMHORIA`).

Connection strings:

- **`ConnectionStrings:Transactions`** – PostgreSQL `masarattransactions` (for tenant resolution).
- **`ConnectionStrings:Wallets`** – PostgreSQL `MasaratWallets`.

RabbitMQ: **`RabbitMQ:Host`**, **`Port`**, **`Username`**, **`Password`**.

## Tenant resolution

See [AML bridge — tenant resolution](aml-bridge-tenant-resolution.md).

## Local / Docker

- **Solution**: `Masarat.Wallet.slnx` includes `src/Masarat.AmlBridge` and `tests/Masarat.AmlBridge.Tests`.
- **Compose**: service `masarat.aml.bridge` wires DB + RabbitMQ + sample `AmlIntegration__BankCodes__*` env vars. Replace the placeholder bank Guid with a real `BankId` from your database before expecting routed traffic in FlowGuard.

## Validation (non-prod)

1. Run wallet stack + bridge.
2. Ensure `aml.transactions` exists on RabbitMQ.
3. Perform a wallet operation that emits a completion event.
4. Confirm FlowGuard analyzer consumes the message.
