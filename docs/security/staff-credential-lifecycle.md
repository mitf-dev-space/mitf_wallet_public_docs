# Staff credential lifecycle (forced password change + credential email)

This page documents how the **management portal** (`Masarat.Gateway.Management.Web`) issues credentials to new staff users and to staff whose passwords are reset by an administrator. It covers the **`MustChangePassword`** flag on the staff identity, the **transactional credential email** that delivers the temporary password, and the **runtime gate** that prevents portal use until the user picks a new password.

This control protects against the most common identity weakness in admin portals: long-lived "first-issued" or admin-chosen passwords that an operator never replaces.

---

## 1. End-to-end flow

```mermaid
flowchart LR
  admin_create[Admin creates user or resets password]
  db_flag[Set MustChangePassword true]
  email[Send SMTP email with login URL plus temporary password]
  login[User signs in]
  gate{MustChangePassword?}
  change[Account ChangePassword]
  app[Blazor staff app]
  admin_create --> db_flag --> email
  login --> gate
  gate -->|yes| change
  gate -->|no| app
  change -->|success clears flag| login
```

1. An admin creates a new portal user, or resets an existing user's password from **Admin → Users**.
2. The portal sets `MustChangePassword = true` on the staff identity row.
3. If outbound email is enabled and the admin keeps the **"Email credentials"** checkbox on, the portal sends a localized email with the sign-in URL and the temporary password.
4. The user signs in. As soon as the cookie is issued, the login page (and the global gate middleware) redirect to **`/Account/ChangePassword`**.
5. The user enters the temporary password as **current password** plus a new password. On success the flag is cleared and the user is signed out and asked to log in with the new password.

The seeded bootstrap admin (`ManagementGateway:SeedAdmin`) is **not** flagged, so dev/local stacks keep working without manual ceremony.

---

## 2. What is enforced where

| Layer | Location | Behavior |
|-------|----------|----------|
| **Persistence** | `AspNetUsers.MustChangePassword` (`boolean`, default `false`) | Source of truth. Default false so existing users stay unaffected. |
| **Admin "Create user"** | Management portal → `/admin/users/create` | Sets the flag, optionally emails credentials, surfaces a warning if the email send fails. |
| **Admin "Reset password"** | Management portal → `/admin/users/{id}` | Sets the flag, optionally emails the new password, surfaces a warning if the email send fails. |
| **Login (password)** | `/Account/Login` | After a successful `PasswordSignInAsync`, redirects to `/Account/ChangePassword?returnUrl=…` when the flag is set. |
| **Login (2FA)** | `/Account/LoginWith2fa` | Same redirect after a successful authenticator code. |
| **Global gate** | `MustChangePasswordMiddleware` (registered after `UseAuthorization`) | Catches all *other* authenticated requests (existing cookie, secondary tab, deep link) and redirects to the change-password page. |
| **Change password** | `/Account/ChangePassword` | On success clears the flag, signs the user out, and returns them to login. Shows a localized banner while the flag is set and hides "Back to dashboard". |

### Path allowlist used by the gate

The gate must let users reach the change-password page itself plus the assets it needs to render. The exact-match list and prefix list are:

- **Exact paths**: `/Account/Login`, `/Account/Logout`, `/Account/LoginWith2fa`, `/Account/AccessDenied`, `/Account/ChangePassword`, `/health`, `/health/ready`, `/manifest.webmanifest`, `/management-theme.js`, `/pwa-register.js`.
- **Prefixes**: `/_framework/`, `/_blazor`, `/_content/`, `/css/`, `/js/`, `/fonts/`, `/branding/`, `/images/`, `/lib/`, `/culture/`.

Everything else under an authenticated session triggers a `302` to `<PathBase>/Account/ChangePassword?returnUrl=<original path + query>`. `PathBase` is honored so reverse-proxy mounts (e.g. `/management`) keep working.

---

## 3. Outbound email

The portal sends a single SMTP email through **MailKit**. The body is HTML + plain-text and contains:

- A localized intro that distinguishes **account creation** from **password reset**.
- The user's email and the temporary password in code-styled boxes.
- A button linking to the **public sign-in URL** built from `OutboundEmail:PublicBaseUrl` + the application's `PathBase` + `/Account/Login`.
- The portal's password policy reminder.
- A brief security note ("contact your administrator if you did not expect this").

### When the service stays silent

- `OutboundEmail:Enabled = false` (default): no message is sent and the service logs an `Information` event. Admins still see the user created / password reset succeed; the optional warning snackbar tells them to share the password manually.
- Required fields missing (`Host`, `FromEmail`): the service logs an `Error` event and returns false; the admin gets the same warning snackbar.
- SMTP connect / authenticate / send failure: logged at `Error` with the exception, warning surfaced in the admin UI.

The portal **never blocks user creation or password reset** when the email cannot be delivered, so admins always retain a manual fallback.

### Configuration

Add these keys to the management portal's configuration (env vars in Docker, sections in `appsettings.json`). All keys live under `ManagementGateway:OutboundEmail`.

| Key | Type | Example | Description |
|-----|------|---------|-------------|
| `Enabled` | `bool` | `true` | Master switch. When `false`, sends are logged and skipped. |
| `Host` | `string` | `smtp.example.com` | SMTP relay hostname. Required when enabled. |
| `Port` | `int` | `587` | SMTP port. |
| `UseStartTls` | `bool` | `true` | Use STARTTLS. When `false`, MailKit auto-negotiates. |
| `Username` | `string?` | `smtp-user` | Optional SMTP auth user. |
| `Password` | `string?` | `smtp-secret` | Optional SMTP auth password. Treat as a secret. |
| `FromEmail` | `string` | `no-reply@masarat.example` | Required envelope/from address. |
| `FromName` | `string?` | `Masarat Management Portal` | Optional display name; falls back to a localized default. |
| `PublicBaseUrl` | `string?` (URL) | `https://management.example.com` | Absolute URL used in email links. Required when the portal is behind a reverse proxy whose host the app cannot infer. Supports a sub-path; combined with the runtime `PathBase`. |

#### Docker Compose snippet

The compose file ships with these keys commented out and disabled by default:

```yaml
environment:
  ManagementGateway__OutboundEmail__Enabled: "false"
  # ManagementGateway__OutboundEmail__Host: "smtp.example.com"
  # ManagementGateway__OutboundEmail__Port: "587"
  # ManagementGateway__OutboundEmail__UseStartTls: "true"
  # ManagementGateway__OutboundEmail__Username: "smtp-user"
  # ManagementGateway__OutboundEmail__Password: "smtp-secret"
  # ManagementGateway__OutboundEmail__FromEmail: "no-reply@masarat.example"
  # ManagementGateway__OutboundEmail__FromName: "Masarat Management Portal"
  # ManagementGateway__OutboundEmail__PublicBaseUrl: "https://management.example.com"
```

### Email content & localization

Subjects, intros, button labels, the password policy line, and the security note are pulled from the management portal resource files (`ManagementWebResource.resx` for English and `ManagementWebResource.ar.resx` for Arabic). The email is rendered in the **admin's UI culture at the time of sending**, which is acceptable for v1; when the recipient's preferred language is recorded in a future iteration, switch to that.

---

## 4. Admin UX

Both admin pages add:

- **Checkbox** "Email credentials to the user" (default **on**). Clear it for break-glass accounts where the admin will deliver the password out of band.
- **Inline hint** explaining that the user will be required to choose a new password on first sign-in.
- **Snackbar warning** when the email send fails (the user / password operation still succeeds).

### Reset password page

In addition to setting the flag, the reset page calls `UserManager.UpdateAsync` and only sends the email after the flag write succeeds, so a user is never emailed a temporary password without the gate being active.

---

## 5. Operational guidance

- **Default posture:** ship with `OutboundEmail:Enabled=false` in any environment that does not have a mail relay; the portal works fine and admins simply share temporary passwords by hand.
- **Production posture:** enable SMTP, set `PublicBaseUrl` to the public hostname terminated by your edge, and verify the email link opens the same login page operators reach normally.
- **Reverse proxies / sub-paths:** if the portal is mounted at e.g. `/management`, set `PublicBaseUrl=https://portal.example.com` (path is optional in the URL because the app appends its own `PathBase`).
- **Auditability:** the existing `management_staff_audit_events` table captures admin actions on the portal. Continue to rely on it for "who reset whom" attribution.
- **Lockout interaction:** lockout state is independent. A lockout on a user with `MustChangePassword=true` still blocks login until the admin clears it; once the user signs in they are immediately routed to the change-password page.
- **2FA interaction:** when 2FA is configured, the gate runs **after** the second factor succeeds. The user proves possession of both their temporary password and their authenticator before being asked to pick a new password.

---

## 6. Database

The portal's Identity database adds one column on the existing **`AspNetUsers`** table:

| Property | PostgreSQL | Default | Description |
|----------|------------|---------|-------------|
| `MustChangePassword` | `boolean`, NOT NULL | `false` | When `true`, the user is forced through `/Account/ChangePassword` before any other portal page renders. |

The migration is **`AddMustChangePasswordToUser`**; it applies on startup via the existing `ManagementDbContext` migration runner. No data backfill is required — historical users keep `false`.

---

## 7. Threats addressed and not addressed

**Addressed**

- Long-lived admin-issued passwords that staff never rotate.
- Casual lateral access by anyone who saw a temporary password being typed in.
- Operator forgetting to deliver credentials by another channel — the email creates an auditable artifact and a clear hand-off.

**Not addressed (out of scope here)**

- **Password mailbox compromise.** Plain-text credentials in email are weaker than a one-time reset link bound to a server token. The current design was chosen for delivery simplicity; revisit if your threat model requires the link-only flow.
- **MFA enrollment.** This page does not enforce that staff enroll an authenticator before first portal use; combine with `ManagementGateway:EnableAuthenticatorEnrollment` and operational policy.
- **Password breach reuse.** ASP.NET Core Identity's password validators (length, classes) are enforced; subscribe to a breached-password feed if your policy requires it.
