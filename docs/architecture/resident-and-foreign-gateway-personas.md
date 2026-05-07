# Residents, foreign holders, and Customer Gateway personas

This page describes how **resident** and **foreign** customers map to the **Customer Gateway** channels **`/v1/customer`** and **`/v1/business`**, and how **KYC** is submitted for the **authenticated user** versus a **managed employee holder**. It reflects the **mitf_wallet** product rules: public onboarding does not create foreign users; foreign identities are **business-managed**; managed-holder KYC is available on the **business** channel only.

For onboarding abuse controls (rate limits, keys, PIN policy), see [Onboarding channel hardening](../security/onboarding-channel-hardening.md). For staff-facing template admin versus runtime apps, see the **Gateway Management Web** service reference and the main repo’s Management Web README.

---

## Terms

| Term | Meaning |
| ---- | ------- |
| **Resident** | `CustomerType.Resident` in **Users**; national ID–oriented identity. |
| **Foreign** | `CustomerType.Foreign` in **Users**; passport / ID number as the primary identifier. |
| **Holder** | The **Users** identity that **owns** the wallet (`UserId` on the wallet aggregate in **Wallets**). |
| **Creator** | The user who initiated wallet creation (`CreatedByUserId`). For employee wallets, this is typically the **business operator**. |
| **Creator management access** | When true, the creator may act on the wallet (list, fund, transactions) and, on **`/v1/business`**, call **managed-holder KYC** for that holder. |

---

## Customer persona — `/v1/customer`

**Who:** Consumer banking / wallet apps (`AppType: customer`).

**Onboarding**

- `POST /v1/customer/onboarding/accounts` drives **Users** onboarding and wallet creation for a **resident** customer.
- **`CustomerType=Foreign` is rejected** for **`OnboardingRegisterCommand`** (HTTP MassTransit onboarding and the same command on the message bus). There is no supported path to “self-register as foreign” through this unified onboarding API.

**Wallets**

- **Self** — holder is the logged-in customer.
- **Relation** — holder is another resident (e.g. family); the creator is the customer who opened the relation.

**KYC (JWT = holder)**

- `POST /v1/customer/kyc/submit` and `GET /v1/customer/kyc/status` always target the **authenticated user** as the KYC subject.
- The active **template key** comes from the wallet **classification** (resident vs foreign template fields on the classification), aligned with the holder type of the wallet being gated.

---

## Business persona — `/v1/business`

**Who:** Corporate operators (`AppType: business`).

**Onboarding**

- `POST /v1/business/onboarding/accounts` registers the **operator** (resident). It does not register employees.

**Employee / sub-wallets**

- `POST /v1/business/wallets` can create wallets whose **holder** is **Resident** or **Foreign** (`holderType`).
- **New foreign holders** are created through **business-managed** registration (e.g. **Users** `RegisterForeign` when the flow provisions a new foreign employee), **not** through `OnboardingRegisterCommand` with `CustomerType=Foreign`.

**KYC**

1. **Self (operator)** — `POST /v1/business/kyc/submit` and `GET /v1/business/kyc/status` use the **JWT user** (the business operator).

2. **Managed holder (employee)** — routes on the **business** channel only:
   - `POST /v1/business/kyc/managed/submit` — body: `holderUserId`, `templateKey`, `values`.
   - `GET /v1/business/kyc/managed/status?holderUserId=...&templateKey=...`

   The gateway authorizes these calls only when the operator has a wallet in their **visible list** where the **holder** matches `holderUserId` and the operator is the **creator with management access** (same visibility rule as listing wallets the operator may act on).

**Template hints (seeded examples in dev)**

- Resident-oriented: `resident-standard`, `resident-employer-issued`.
- Foreign-oriented: `foreign-standard`, `foreign-employer-issued`.

Exact keys depend on **Wallet** classification configuration and **KYC** template definitions in your environment.

---

## Why the split exists

- **Compliance:** foreign identities are tied to **employer / programme context** (classification, templates, review), not anonymous self-service onboarding.
- **Authorization:** Money movement already enforces **actor access** to wallets; managed KYC reuses the same **creator + management access** notion so operators cannot submit KYC for arbitrary user IDs.

---

## Related material

| Topic | Where |
| ----- | ----- |
| Customer Gateway host and headers | [Customer Gateway service reference](../reference/service-reference/Masarat.Gateway.Customer.Api.reference.md) |
| Onboarding channel controls | [Onboarding channel hardening](../security/onboarding-channel-hardening.md) |
| Users HTTP / behaviour | [Users API service reference](../reference/service-reference/Masarat.Users.Api.reference.md) |
| KYC gRPC surface | [KYC API service reference](../reference/service-reference/Masarat.Kyc.Api.reference.md) |
| In-repo narrative (developers) | **mitf_wallet** — `src/Masarat.Gateway/Masarat.Gateway.Customer.Api/personas-and-flows.md`, `docs/solutions/kyc-user-status-vs-template.md` |

The **mitf_wallet** repo also ships **`mobile/gateway_tester`**, a Flutter harness with guided flows that include self KYC and **managed-holder** KYC steps on the business track.
