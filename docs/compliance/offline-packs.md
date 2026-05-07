# Offline compliance packs (print & PDF)

Print or export **Security** and **Reconciliation** chapters for offline audit binders. No pre-built PDFs ship; generate them from the live HTML using your browser's **Print → Save as PDF**.

---

## Quick method

1. Open the page in the published site or a local `mkdocs serve` build.
2. **Ctrl+P** / **⌘P** → **Save as PDF**.
3. Enable **Background graphics** if diagrams or code blocks must match the on-screen theme.

Material hides navigation chrome when printing — single-column output.

---

## Security (print these pages)

| Topic | Page |
| ----- | ---- |
| System hardening (API keys, PINs, tokens, ops) | [System hardening](../security/system-hardening.md) |
| Onboarding channel | [Onboarding channel hardening](../security/onboarding-channel-hardening.md) |

---

## Reconciliation (print these pages)

| Topic | Page |
| ----- | ---- |
| Financial operations narrative | [Financial operations](../reconciliation/financial-operations-and-reconciliation.md) |
| Reconciliation job | [Reconciliation job](../reconciliation/reconciliation.md) |

---

## Optional: operations runbook (often requested with reconciliation)

| Topic | Page |
| ----- | ---- |
| Reconciliation & consistency runbook | [Runbook](../operations/reconciliation-and-consistency-runbook.md) |

---

## Evidence trail

- **Last updated** and **version badge** appear at the bottom of each page (git dates + docs bundle label).
- **Change stream:** [Changelog & releases](../changelog.md); after each deploy, syndication files live at the site root as `feed_rss_updated.xml` and `feed_json_updated.json` (also linked from the top banner).

If you need **official** stamped PDFs from Masarat, track that through your programme office or account team and reference the export timestamp and site version badge in your audit workpapers.
