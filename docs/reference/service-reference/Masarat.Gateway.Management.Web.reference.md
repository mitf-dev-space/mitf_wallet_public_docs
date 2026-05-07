# Masarat.Gateway.Management.Web — configuration and database

Connection string name: **`Management`** → database commonly `MasaratManagement`.  
Hosts **ASP.NET Core Identity** (staff users) + **`management_staff_audit_events`**.

---

## Configuration

| Key | Value type | Example value | Description |
|-----|------------|---------------|-------------|
| `Logging:LogLevel:*` | `string` | — | Standard logging. |
| `AllowedHosts` | `string` | `*` | Host filter. |
| `ConnectionStrings:Management` | `string` | Npgsql | Identity + audit DB. |
| `ManagementGateway:DefaultBankId` | `Guid` (string) | all-zero + `...0001` | Default `x-bank-id` for downstream gRPC. |
| `ManagementGateway:Downstream:UsersGrpcAddress` | `string` (URI) | `http://localhost:5003` | Users gRPC. |
| `ManagementGateway:Downstream:WalletGrpcAddress` | `string` (URI) | `http://localhost:5002` | Wallets gRPC. |
| `ManagementGateway:Downstream:TransactionsGrpcAddress` | `string` (URI) | `http://localhost:5004` | Transactions gRPC. |
| `ManagementGateway:Downstream:LedgerGrpcAddress` | `string` (URI) | `http://localhost:5001` | Ledger gRPC. |
| `ManagementGateway:Downstream:KycGrpcAddress` | `string` (URI) | `http://localhost:5008` | KYC gRPC. |
| `ManagementGateway:Downstream:ReconciliationReportingBaseUrl` | `string` (URI) | `http://localhost:5005` | Reporting HTTP API. |
| `ManagementGateway:Downstream:DownstreamApiKey` | `string?` | — | Optional key for downstream calls. |
| `ManagementGateway:DownstreamResilience:ReadTimeoutMs` | `int` | `15000` | Default read timeout. |
| `ManagementGateway:DownstreamResilience:TransactionsReadTimeoutMs` | `int` | `120000` | Heavy transaction reads. |
| `ManagementGateway:DownstreamResilience:ReconciliationReadTimeoutMs` | `int` | `180000` | Reporting/export reads. |
| `ManagementGateway:DownstreamResilience:WriteTimeoutMs` | `int` | `30000` | gRPC write timeout. |
| `ManagementGateway:DownstreamResilience:HttpWriteTimeoutMs` | `int` | `15000` | HTTP write timeout. |
| `ManagementGateway:DownstreamResilience:MaxRetryAttempts` | `int` | `2` | Retries. |
| `ManagementGateway:DownstreamResilience:BaseDelayMs` | `int` | `150` | Backoff. |
| `ManagementGateway:DownstreamResilience:BreakDurationSeconds` | `int` | `20` | Circuit breaker. |
| `ManagementGateway:DownstreamResilience:MinimumThroughput` | `int` | `8` | Breaker sampling. |
| `ManagementGateway:DownstreamResilience:FailureRatio` | `double` | `0.5` | Breaker threshold. |
| `ManagementGateway:SeedAdmin:Enabled` | `bool` | `false` | Seed initial admin on startup. |
| `ManagementGateway:SeedAdmin:Email` | `string?` | — | Admin email when seeding. |
| `ManagementGateway:SeedAdmin:Password` | `string?` | — | Admin password when seeding. |
| `ManagementGateway:OutboundEmail:Enabled` | `bool` | `false` | Master switch for SMTP credential emails. When `false`, sends are skipped and logged. |
| `ManagementGateway:OutboundEmail:Host` | `string?` | `smtp.example.com` | SMTP relay hostname (required when enabled). |
| `ManagementGateway:OutboundEmail:Port` | `int` | `587` | SMTP port. |
| `ManagementGateway:OutboundEmail:UseStartTls` | `bool` | `true` | Use STARTTLS; when `false` MailKit auto-negotiates. |
| `ManagementGateway:OutboundEmail:Username` | `string?` | — | Optional SMTP auth user. |
| `ManagementGateway:OutboundEmail:Password` | `string?` | — | Optional SMTP auth password (treat as a secret). |
| `ManagementGateway:OutboundEmail:FromEmail` | `string?` | `no-reply@masarat.example` | Required envelope/from address when enabled. |
| `ManagementGateway:OutboundEmail:FromName` | `string?` | `Masarat Management Portal` | Optional display name; falls back to a localized default. |
| `ManagementGateway:OutboundEmail:PublicBaseUrl` | `string?` (URL) | `https://management.example.com` | Absolute base URL used in email sign-in links. Combined with the runtime `PathBase`. |
| `ManagementGateway:TrustForwardedHeaders` | `bool` | `true` | Honor `X-Forwarded-*`. |
| `ManagementGateway:TrustedForwardedProxies` | `string[]` | `[]` | Known proxy IPs. |
| `ManagementGateway:TrustedForwardedNetworks` | `string[]` | `[]` | Trusted CIDRs (e.g. `10.0.0.0/8`). |
| `ManagementGateway:UseHttpsRedirection` | `bool` | `false` | HTTP→HTTPS redirect. |
| `ManagementGateway:EnableAuthenticatorEnrollment` | `bool` | `true` | TOTP enrollment UI. |
| `ManagementGateway:HelpDocumentationBaseUrl` | `string?` | URL | Help page doc link. |
| `ManagementGateway:HelpRunbookLinks` | `array` of `{ Title, Url }` | — | Extra runbook links on Help page. |
| `Observability:*` | various | — | Telemetry toggles (many disabled by default in sample). |

---

## Database tables

### ASP.NET Core Identity (Guid PK)

Standard tables (see EF migrations snapshot): **`AspNetUsers`**, **`AspNetRoles`**, **`AspNetUserRoles`**, **`AspNetUserClaims`**, **`AspNetRoleClaims`**, **`AspNetUserLogins`**, **`AspNetUserTokens`**.

**`AspNetUsers`** (custom `ManagementUser`) — main columns:

| Property | Type | PostgreSQL | Description |
|----------|------|------------|-------------|
| `Id` | `Guid` | `uuid`, PK | User id. |
| `UserName` / `NormalizedUserName` | `string` | `varchar(256)` | Login name. |
| `Email` / `NormalizedEmail` | `string?` | `varchar(256)` | Email. |
| `EmailConfirmed` / `PhoneNumberConfirmed` / `TwoFactorEnabled` | `bool` | `boolean` | Flags. |
| `PasswordHash` / `SecurityStamp` / `ConcurrencyStamp` | `string` | text | Credentials & concurrency. |
| `PhoneNumber` | `string?` | text | Phone. |
| `LockoutEnabled` / `LockoutEnd` / `AccessFailedCount` | bool / DateTimeOffset? / int | — | Lockout. |
| `MustChangePassword` | `bool` | `boolean`, default `false` | When `true`, the user is forced through `/Account/ChangePassword` before any other portal page renders. Set when an admin creates a user or resets a password; cleared when the user successfully changes it. See [Staff credential lifecycle](../../security/staff-credential-lifecycle.md). |

Other Identity tables follow Microsoft’s default column shapes (`RoleId`, `UserId`, `ClaimType`, `ClaimValue`, etc.).

### `management_staff_audit_events`

| Property | Type | PostgreSQL | Description |
|----------|------|------------|-------------|
| `Id` | `Guid` | `uuid`, PK | Event id. |
| `StaffUserId` | `Guid` | `uuid` | Staff user who acted. |
| `StaffEmail` | `string` | `varchar(256)` | Email snapshot. |
| `Action` | `string` | `varchar(128)` | Action verb (e.g. export, suspend). |
| `EntityType` | `string` | `varchar(64)` | Domain type. |
| `EntityId` | `string` | `varchar(256)` | Target id string. |
| `Details` | `string?` | text | Optional JSON / text payload. |
| `CreatedAtUtc` | `DateTime` | `timestamptz` | When recorded. |
