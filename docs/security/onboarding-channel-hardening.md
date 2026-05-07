# Onboarding channel hardening

`POST .../onboarding/accounts` is an **unauthenticated** (no user JWT) channel protected only by **app credentials** (`X-App-Id`, `X-App-Key`) and Users service enforcement. Optional `pin` in the same request sets the first wallet PIN after provisioning via a gateway-trusted actor on the Wallets gRPC call — there is no public "set first PIN by `walletId` alone" HTTP route.

These controls are **not automatic** — operators and product must configure them.

## 1. Rate limiting and abuse

- **Users service:** Configure `OnboardingRateLimiting` (`PermitLimit`, `WindowSeconds`) on `Masarat.Users.Api` for `POST /onboarding/accounts`. Tune per environment; stricter in production.
- **Gateway:** Apply partitioned rate limits (`CustomerGateway:RateLimiting`).
- **WAF / edge:** For public internet deployments, add CDN or API gateway throttles and bot management.

## 2. Identity and eligibility

- **Resident vs foreign:** Unified Users onboarding is **resident-only**. `CustomerType=Foreign` is rejected; foreign holders are introduced through business-managed registration. See [Retail vs corporate: resident & foreign customers](../architecture/resident-and-foreign-gateway-personas.md).
- **National ID / KYC:** Strengthen downstream verification (core banking, national registry, document checks). The gateway only forwards profile fields.
- **Duplicate handling:** Rely on Users 409 Conflict and clear client UX; avoid ambiguous retries.
- **Linked bank account:** When `linkedBankAccountId` is required by policy, enforce it in Users or bank validation, not only in the mobile app.

## 3. App keys and transport

- **Rotate `X-App-Key`** on compromise; treat keys as confidential — no logging, no client-side bundling for untrusted builds.
- **TLS:** Terminate at the edge; do not expose onboarding over cleartext in production.
- **Attestation (optional):** For high-risk retail apps, consider device attestation.

## 4. Observability and response

- **Log** onboarding outcomes with correlation id; alert on spikes in failures, 429s, or `pinSetupError` rates.
- **Runbooks:** Define steps when `pinConfigured: false` after onboarding.

## 5. PIN policy alignment

`CustomerGateway:OnboardingWalletPin` (`MinLength`, `MaxLength`) must match **Wallets** `WalletPin` settings to avoid rejections downstream after gateway validation passes.

## 6. What this does not replace

- Strong identity proof (OTP, bank-signed assertion, in-branch step) if regulators or fraud models require it.
- PIN recovery (forgotten-PIN flow, branch reset) — out of scope for this API note but required for a complete product.

For PIN storage, lockout, and gRPC semantics, see [system-hardening.md](./system-hardening.md).
