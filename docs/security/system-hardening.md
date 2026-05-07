# System Hardening and Security

API key authentication, wallet PIN and transaction authorization, logging controls, and operational recommendations.

---

## 1. API key authentication

When enabled, every request (REST and gRPC) must include a valid key — otherwise rejected with **401 Unauthenticated**.

**Applies to:** Users API (5003), Ledger API (5001), Wallets API (5002), Transactions API (5004).
- **REST:** `ApiKeyMiddleware`
- **gRPC:** `ApiKeyGrpcInterceptor`

**Bypass:** `GET /health` and `GET /health/ready` are always allowed without a key.

### Sending the key

| Channel | Header / metadata | Example |
| ------- | ----------------- | ------- |
| REST | `X-Api-Key: <key>` | `curl -H "X-Api-Key: your-key" ...` |
| REST | `Authorization: ApiKey <key>` | `Authorization: ApiKey your-key` |
| gRPC | Metadata `x-api-key` | `grpcurl -H "x-api-key: your-key" ...` |
| gRPC | Metadata `authorization: ApiKey <key>` | Same as REST style |

Key comparison is **ordinal** (case-sensitive).

### Configuration

| Key | Description | Default |
| --- | ----------- | ------- |
| `Auth:ApiKey` | Shared secret key | `""` |
| `Auth:RequireApiKey` | When `true`, require a valid key on every request (except health) | `true` |

**Production:** Use a strong, random key (32+ chars); keep `RequireApiKey: true`; never commit to source control.

---

## 2. Wallet PIN and transaction authorization

Optional wallet PIN and short-lived transaction authorization tokens for debit operations.

- **Wallet PIN** — 4–6 digit PIN per wallet, stored as PBKDF2 hash with per-wallet salt.
- **Transaction authorization token** — short-lived token (default 5 min) issued by Wallets API after successful `VerifyWalletPin`. Required by Transactions API on debit RPCs when the wallet classification's `OperationAuthMode` is **user PIN**; not required for **external OTP trusted session** classifications.

### Flow

1. Set PIN via `POST .../onboarding/accounts` (optional `pin` field) or `SetWalletPin` gRPC.
2. Before a debit: call `VerifyWalletPin` → get `transaction_authorization_token`.
3. Call debit RPC (Transfer, FundWallet, ProcessMerchantPayment, ProcessCashWithdrawal) with the token.
4. Transactions API validates: signature, expiry, and wallet match.

### Wallets API RPCs (port 5002)

| RPC | Description |
| --- | ----------- |
| `SetWalletPin` | Set or overwrite PIN. Requires user actor context when `x-bank-id` is present. Digits only, 4–6 chars. |
| `ChangeWalletPin` | Change PIN; requires current PIN. |
| `VerifyWalletPin` | Verify PIN; returns `transaction_authorization_token`. Failed attempts increment lockout counter; success resets it. |

**PIN rules (configurable via `WalletPin`):**
- `MinPinLength` / `MaxPinLength` (default 4–6 digits)
- Digits only (0–9)
- `MaxFailedAttempts` (default 5) → lockout for `LockoutMinutes` (default 15)

### Transaction requests (Transactions API — port 5004)

Optional `transaction_authorization_token` on `TransferRequest`, `FundWalletRequest`, `ProcessMerchantPaymentRequest`, `ProcessCashWithdrawalRequest`. Required when wallet classification requires user PIN.

If a token is required but `Secret` is empty: *"Transaction authorization is not configured."*

### Configuration

**Wallets API:**

```json
{
  "WalletPin": {
    "MaxFailedAttempts": 5,
    "LockoutMinutes": 15,
    "MinPinLength": 4,
    "MaxPinLength": 6
  },
  "WalletAuthorizationToken": {
    "Secret": "change-me-in-production-use-strong-secret",
    "ExpiryMinutes": 5
  }
}
```

**Transactions API:**

```json
{
  "WalletAuthorizationToken": {
    "Secret": "change-me-in-production-use-strong-secret",
    "ExpiryMinutes": 5
  }
}
```

### Security details

**PIN storage:**
- Hashed with PBKDF2-HMAC-SHA256: 100,000 iterations, 32-byte random salt per wallet, 32-byte hash.
- Timing-safe comparison (`CryptographicOperations.FixedTimeEquals`).

**Token:**
- HMAC-SHA256 signed. Payload: version byte (1) + wallet ID (16 bytes) + expiry (UTC ticks, 8 bytes).
- Format: `base64url(payload).base64url(signature)`.
- Transactions API validates: token present, signature, expiry, wallet match (timing-safe).

**Dual signing (future):** The same flow will support dual signing — token issued only after PIN + second factor (e.g. mobile app approval). The Transactions API contract stays unchanged.

---

## 3. Logging and sensitive data

When `Observability:ApiLogging:EnableRequestLogging` is enabled:

- **Headers:** `LogRequestHeaders=true` → sensitive headers redacted (`Authorization`, `Cookie`, `X-Api-Key` → `[REDACTED]`).
- **Body:** `LogRequestBody` and `LogResponseBody` default to `false`. Do not enable in production.
- **ExcludePaths:** `/health`, `/health/ready`, `/metrics`, `/hc` excluded by default.

```json
{
  "Observability": {
    "ApiLogging": {
      "EnableRequestLogging": false,
      "LogRequestHeaders": false,
      "LogRequestBody": false,
      "LogResponseBody": false,
      "RedactHeaders": ["Authorization", "Cookie", "X-Api-Key"],
      "RedactBodyProperties": ["password", "pin", "apiKey", "token", "authorization"],
      "ExcludePaths": ["/health", "/health/ready", "/metrics", "/hc"]
    }
  }
}
```

**General practices:**
- Never log API keys, PINs, or transaction authorization tokens.
- `X-Correlation-ID` is safe to log.

---

## 4. Bank context (x-bank-id)

Transaction and wallet RPCs acting on behalf of a bank require the `x-bank-id` header (gRPC metadata) set to the calling bank's GUID. This is an **authorization context** — ensure only trusted clients call these APIs with the correct bank ID.

---

## 5. Operational recommendations

| Area | Recommendation |
| ---- | -------------- |
| **TLS** | HTTPS/TLS for all API and gRPC traffic; terminate at reverse proxy |
| **Secrets** | Environment variables or vault; never in source control |
| **API key** | `RequireApiKey: true`; strong `Auth:ApiKey` on all services |
| **Wallet PIN secret** | Strong `WalletAuthorizationToken:Secret` (e.g. 256-bit); same value in Wallets and Transactions |
| **Health endpoints** | Unauthenticated by design for probes; no sensitive operations on these paths |
| **Swagger / OpenAPI** | Disable or restrict in production |
| **Network** | Restrict network access to APIs and databases; mTLS or API key for service-to-service |

For runbooks and log queries: [Logging](../operations/logging.md). For API and gRPC details: [API reference](../reference/api.md).
