# KYC flow, validation, and portal approval

Covers **field definitions**, **templates**, **submissions**, **validation**, gateway coupling, and **staff approval** in Gateway Management Web.

For which programme collects KYC (consumer vs corporate), see [Resident and foreign customers](./resident-and-foreign-gateway-personas.md). For `KycStatus` vs per-template state, see `docs/solutions/kyc-user-status-vs-template.md`.

---

## Two surfaces

| Surface | Role | Users |
| ------- | ---- | ----- |
| **Customer Gateway** (`Masarat.Gateway.Customer.Api`) | Collects field values from mobile / partner apps; calls KYC gRPC with the holder user id | Customers, business operators |
| **Gateway Management Web** (`Masarat.Gateway.Management.Web`) | Configures templates and field definitions; lists pending submissions; approves or rejects when manual review is required | Bank Viewer (read), Operator / Admin (mutations + review) |

Apps **submit** and **poll status** — they do not approve. The portal is the human review lane.

---

## Data model

### Field definitions (catalog)

- Global dictionary of fields: **key**, label, **data type** (`String`, `Date`, `Number`, `Boolean`), and **sensitive** flag.
- Managed at `/kyc-fields`. New fields are added once; multiple templates reuse the same key.

### Templates (product-specific shape)

- A **template** has a stable **key** (e.g. `resident-standard`), display metadata, **`RequiresManualReview`**, **`IsActive`**, and field links with **required** / optional flags and **sort order**.
- Managed at `/kyc-templates`.

**Why "dynamic":** Wallet **classifications** point wallet products at resident vs foreign template keys and a **KYC capture mode**. When templates or classifications change, apps see different `templateKey` requirements without redeployment.

### Submissions (runtime state)

- One submission row per **(holder user id, template id)** in the KYC service.
- Stores **merged** field values across partial submits, **status**, and review metadata.

---

## End-to-end flow

```mermaid
sequenceDiagram
  participant App as Customer / business app
  participant GW as Customer Gateway
  participant KYC as KYC service
  participant Staff as Management Web

  App->>GW: POST .../kyc/submit (or /kyc/managed/submit)
  GW->>KYC: SubmitKycFields (holder, templateKey, values)
  KYC-->>GW: success, submissionId, status
  GW-->>App: JSON body

  App->>GW: GET .../kyc/status
  GW->>KYC: GetKycSubmissionStatus
  KYC-->>GW: missingKeys, cleared flag, template snapshot
  GW-->>App: drives UI / wallet creation messaging

  alt Template requires manual review
    Staff->>KYC: ListPendingKycSubmissions / GetKycSubmissionDetail
    Staff->>KYC: ReviewKycSubmission (approve or reject)
    Note over App,KYC: After approval, status shows cleared when rules satisfied
  end
```

---

## Submission lifecycle

| Status | Meaning |
| ------ | ------- |
| **Pending** | Values may be partial or complete. If `RequiresManualReview` is true, auto-approval never runs — stays Pending until staff review. |
| **Approved** | Auto-approved (all required fields present + no manual review required) or approved by staff. |
| **Rejected** | Staff rejected; reason stored for audit. |

Only **Pending** submissions can receive `ReviewKycSubmission`; terminal states return an error.

---

## Validation on submit

When **SubmitKycFields** runs:

1. **Template key** — non-empty; template must exist and be **active**.
2. **Field keys** — every submitted key must belong to the template's linked fields.
3. **First save** — at least one value required when no row exists yet.
4. **Merge** — new values overlay prior stored values.
5. **Data types** — validated per field definition (number, date, boolean).
6. **Required completeness** — after merge, all required fields must be non-whitespace to auto-approve (when manual review is off).

Failure returns `success: false` and an `error` string.

---

## Status query

**`GET .../kyc/status`** returns:

- **`template`** snapshot — fields, types, required flags, `isActive`.
- **`missingFieldKeys`** / **`presentFieldKeys`** — derived from required flags vs stored values.
- **`kycCleared`** — `true` only when submission is **Approved** and template is still **active**. Inactive templates never clear KYC, even if a prior row was approved.

---

## Gateway coupling to wallets

When the gateway creates a wallet, it resolves the **KYC template key** from the wallet classification and calls the KYC service.

**`WalletKycCaptureMode`** controls strictness:

- **`RequireAtCreation`** — missing or uncleared KYC blocks wallet creation (`missing_kyc`).
- **Deferred modes** — wallet can be created with a `pending_kyc` state; user finishes KYC later.

The gateway also treats the Users `KycStatus` field as a shortcut: if already `Approved`, the resolver may treat KYC as complete without calling the KYC service.

---

## Portal: where staff work

All routes are on **Gateway Management Web** (Blazor). KYC review mutations require **Operator** or **Admin**.

| Route | Purpose |
| ----- | ------- |
| **`/kyc-review`** | Queue of pending submissions; open detail; Approve or Reject. Sensitive field values are masked. |
| **`/kyc-templates`** | List/create/edit templates — keys, manual review flag, active flag, linked fields. |
| **`/kyc-fields`** | Maintain global field definitions (keys, labels, data types, sensitive flag). |

Approve/reject actions append **staff audit** events for later investigation.

---

## Integrator quick reference

| Goal | Call |
| ---- | ---- |
| Discover required fields | `GET .../kyc/status?templateKey=...` (or managed variant with `holderUserId`) |
| Save partial progress | `POST .../kyc/submit` with a subset of keys; repeat until `missingFieldKeys` is empty |
| Know if wallet can proceed | Use gateway wallet-creation response — depends on `kycCleared` and capture mode |
| Staff decision | Use Management Web `/kyc-review`, not a public REST route on the customer gateway |

---

## Related pages

| Topic | Link |
| ----- | ---- |
| Retail vs corporate KYC entry points | [Resident and foreign customers](./resident-and-foreign-gateway-personas.md) |
| KYC gRPC reference | [KYC API service reference](../reference/service-reference/Masarat.Kyc.Api.reference.md) |
| Management portal host | [Gateway Management Web service reference](../reference/service-reference/Masarat.Gateway.Management.Web.reference.md) |
| Onboarding and abuse | [Onboarding channel hardening](../security/onboarding-channel-hardening.md) |
