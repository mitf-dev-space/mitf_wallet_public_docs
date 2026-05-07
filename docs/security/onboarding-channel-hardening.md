# Onboarding channel hardening

Customer and business **`POST …/onboarding/accounts`** is an **unauthenticated** (no user JWT) channel protected only by **app credentials** (`X-App-Id`, `X-App-Key`, etc.) and whatever the **Users** service enforces. Optional **`pin`** in the same request sets the first wallet PIN **after** provisioning, using a **gateway-trusted** actor on the Wallets gRPC call—there is no public “set first PIN by `walletId` alone” HTTP route.

This document lists controls that **do not ship automatically** in code but operators and product should apply to address residual risk (weak onboarding, abuse, leaked app keys).

## 1. Rate limiting and abuse

- **Users service:** Configure **`OnboardingRateLimiting`** (`PermitLimit`, `WindowSeconds`) on **Masarat.Users.Api** for **`POST /onboarding/accounts`** so automated account farming is expensive. Tune per environment; stricter in production.
- **Gateway:** The gateway applies **partitioned rate limits** (`CustomerGateway:RateLimiting`). Ensure onboarding-class traffic is not given an overly generous bucket if you add route-specific tuning later.
- **WAF / edge:** For public internet deployments, add CDN or API gateway throttles and bot management in front of the customer gateway.

## 2. Identity and eligibility

- **Resident vs foreign (product boundary):** Unified **Users** onboarding (`OnboardingRegisterCommand`, including gateway `POST …/onboarding/accounts`) is **resident-only**. **`CustomerType=Foreign` is rejected** so foreign end users cannot be created through that path; foreign holders are introduced through **business-managed** registration (e.g. when creating a **business employee** wallet with a foreign holder). See [Retail vs corporate: resident & foreign customers](../architecture/resident-and-foreign-gateway-personas.md) for the business-facing summary and links to technical detail.
- **National ID / KYC:** Strengthen **downstream** verification (core banking, national registry, document checks) so “onboarding as someone else” is not trivial. The gateway only forwards profile fields; trust boundaries live in **Users** and bank integrations.
- **Duplicate handling:** Rely on Users **409 Conflict** and clear client UX when an identity already exists; avoid ambiguous retries that create support load.
- **Linked bank account:** When **`linkedBankAccountId`** is required by policy, enforce it in **Users** or bank validation, not only in the mobile app.

## 3. App keys and transport

- **Rotate `X-App-Key`** on compromise; treat keys as **confidential** (no logging, no client-side bundling for untrusted builds).
- **TLS:** Terminate TLS at the edge; do not expose onboarding over cleartext in production.
- **Attestation (optional):** For high-risk retail apps, consider device attestation or similar to bind onboarding to genuine app builds.

## 4. Observability and response

- **Log** onboarding outcomes with **correlation id**; alert on spikes in failures, 429s, or **`pinSetupError`** rates from the gateway response.
- **Runbooks:** Define steps when **`pinConfigured`: false** after onboarding (support reset, manual reconciliation, or guided retry once wallet is provisioned).

## 5. PIN policy alignment

- Gateway validates optional onboarding PIN using **`CustomerGateway:OnboardingWalletPin`** (`MinLength`, `MaxLength`). Keep these aligned with **Wallets** **`WalletPin`** settings so users are not rejected downstream after passing gateway validation.

## 6. What merged onboarding + PIN does *not* replace

- **Strong identity proof** (OTP to phone, bank-signed assertion, in-branch step) if regulators or fraud models require it.
- **Recovery** when the user forgets the PIN (forgotten-PIN flow, branch reset)—out of scope for this API note but required for a complete product.

For PIN storage, lockout, and gRPC semantics, see [system-hardening.md](./system-hardening.md).
