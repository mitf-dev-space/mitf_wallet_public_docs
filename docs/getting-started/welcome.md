# Welcome — how to use this site

Pick your lane and follow the **reading order** so nothing depends on missing context.

## Choose your path

| If you are… | Read first | Then |
| ----------- | ---------- | ---- |
| **Building a mobile / partner integration** | [5-minute quickstart](../getting-started/quickstart.md), [API reference](../reference/api.md), [gRPC services](../reference/grpc-services.md) | [Retail vs corporate](../architecture/resident-and-foreign-gateway-personas.md), [Transfer backpressure](../architecture/transfer-backpressure-client-contract.md), [Domain events](../architecture/events.md) |
| **Running production** | [Production deployment](../operations/production-deployment.md), [Logging](../operations/logging.md) | [Reconciliation runbook](../operations/reconciliation-and-consistency-runbook.md), [Outbox contract](../architecture/outbox-and-ledger-consistency.md) |
| **Tuning config / env** | [Configuration reference](../reference/configuration-reference.md) | [Service reference index](../reference/service-reference/README.md) |
| **KYC product & staff review** | [KYC flow, validation & portal approval](../architecture/kyc-flow-validation-and-portal-approval.md) | [KYC API ref](../reference/service-reference/Masarat.Kyc.Api.reference.md), [Management Web ref](../reference/service-reference/Masarat.Gateway.Management.Web.reference.md) |
| **Finance / AML oversight** | [Financial operations](../reconciliation/financial-operations-and-reconciliation.md) | [Reconciliation job](../reconciliation/reconciliation.md), [FlowGuard plan](../integrations/flowguard-wallet-aml.md) |

---

## Site structure

Folders are **topic-based**:

| Folder | Content |
| ------ | ------- |
| `stakeholders/` | Leadership-oriented overviews |
| `getting-started/` | Quickstart, site map, customer segments |
| `architecture/` | Events, consistency, flows, personas, KYC |
| `operations/` | Deploy, logging, runbooks, load testing |
| `security/` | API keys, PIN, hardening, staff credentials |
| `reconciliation/` | Finance narrative, daily job |
| `integrations/` | AML bridge and FlowGuard plan |
| `load-testing/` | Reference runs and operator guide |
| `reference/` | API, gRPC, config, per-service refs |

---

## Diagram types

| Diagram | Typical use |
| ------- | ----------- |
| **Flowchart** | Component maps, choose-your-path |
| **Sequence** | Request / event timelines |
| **State** | Async delivery and status models |

When a page is dense, skim the **mermaid diagrams** first — they compress the narrative.

---

- [Full A–Z list of every page](all-pages.md)
- [Repository README](https://github.com/anstwechy/mitf_wallet_public_docs/blob/main/README.md) — local build & GitHub Pages
