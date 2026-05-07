# Welcome — how to use this site

Pick your lane below, then follow the **reading order** so nothing depends on missing context.

!!! tip "New here?"
    **Masarat leadership:** start with [Stakeholder hub](../stakeholders/index.md). **Integrators, operators, and engineers:** use [Welcome & guided tours](welcome.md) (reading order and paths); the [site home](../README.md) is a short Masarat-oriented welcome.

---

## Choose your path

```mermaid
flowchart LR
  subgraph roles["Who are you?"]
    I([Integrator])
    O([Operator])
    C([Config or SRE])
    F([Finance / audit])
  end

  I --> A1[API + gRPC]
  A1 --> A2[Backpressure contract]
  A2 --> A3[Events / async]

  O --> B1[Deploy + logs]
  B1 --> B2[Runbooks]
  B2 --> B3[Load tests]

  C --> C1[Config reference]
  C1 --> C2[Per-service refs]

  F --> D1[Finance narrative]
  D1 --> D2[Reconciliation job]

  A1 -.-> R1["reference/api.md"]
  A2 -.-> R2["architecture/transfer-backpressure..."]
  B1 -.-> R3["operations/production-deployment.md"]
  C1 -.-> R4["reference/configuration-reference.md"]
  D1 -.-> R5["reconciliation/financial-operations..."]
```

| If you are… | Read first | Then |
| ----------- | ---------- | ---- |
| **Building a mobile / partner integration** | [5-minute quickstart](../getting-started/quickstart.md), [API reference](../reference/api.md), [gRPC services](../reference/grpc-services.md) | [Residents / foreign / personas](../architecture/resident-and-foreign-gateway-personas.md), [Transfer backpressure](../architecture/transfer-backpressure-client-contract.md), [Domain events](../architecture/events.md) |
| **Running production** | [Production deployment](../operations/production-deployment.md), [Logging](../operations/logging.md) | [Reconciliation runbook](../operations/reconciliation-and-consistency-runbook.md), [Outbox contract](../architecture/outbox-and-ledger-consistency.md) |
| **Tuning YAML / env** | [Configuration reference](../reference/configuration-reference.md) | [Service reference index](../reference/service-reference/README.md) |
| **Finance / AML oversight** | [Financial operations](../reconciliation/financial-operations-and-reconciliation.md) | [Reconciliation job](../reconciliation/reconciliation.md), [FlowGuard plan](../integrations/flowguard-wallet-aml.md) |

---

## How the bookshelf is organized

Folders are **topic-based** (not alphabet soup):

```mermaid
flowchart TB
  subgraph lead["Masarat leadership"]
    ST[stakeholders/]
  end

  subgraph start["First steps"]
    GS[getting-started/]
    SH[Site home]
    W[Welcome & tours]
  end

  subgraph depth["Technical depth"]
    A[architecture]
    O[operations]
    S[security]
    RC[reconciliation]
    I[integrations]
    L[load-testing]
    R[reference]
  end

  ST --> SH
  GS --> SH
  SH --> W
  W --> A
  W --> O
  W --> S
  W --> RC
  W --> I
  W --> L
  W --> R
```

---

## Diagram types you will see

| Diagram | Typical use in these docs |
| ------- | ------------------------- |
| **Flowchart** (`flowchart`) | Choose-your-path, component maps |
| **Sequence** (`sequenceDiagram`) | Request / event timelines across services |
| **State / timeline** | Async money movement and delivery guarantees |

When a page is dense, skim its **mermaid** first — it is the compressed version of the narrative.

---

## Next steps

- [Full A–Z list of every page](all-pages.md)
- [Repository README](https://github.com/anstwechy/mitf_wallet_public_docs/blob/main/README.md) — local build & GitHub Pages
