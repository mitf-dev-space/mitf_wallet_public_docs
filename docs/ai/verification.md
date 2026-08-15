# MITF Wallet Public Docs — Cloud Agent Verification

Audit date: 2026-08-15  
Repository: `mitf-dev-space/mitf_wallet_public_docs`

## Purpose

Public MkDocs site for Masarat Wallet documentation. Published to GitHub Pages.

## Branches

| Branch | Role |
|--------|------|
| `main` | Default; Pages deploy target |

## Runtimes

| Component | Version |
|-----------|---------|
| Python | **3.12** in CI (`actions/setup-python`); use 3.11–3.12 locally |
| MkDocs Material | `9.5.49` (pinned in `requirements.txt`) |
| MkDocs | `1.6.1` |

## Manifest inventory

| Artifact | Path |
|----------|------|
| `requirements.txt` | repo root |
| `mkdocs.yml` | repo root |
| CI | `.github/workflows/pages.yml` |

No Docker, .NET, or npm manifests.

## Docker / Compose

None for doc build.

## Verified safe commands (2026-08-15)

Run from repo root:

```bash
python -m venv .venv
# Windows: .\.venv\Scripts\activate
pip install -r requirements.txt
mkdocs build --strict
```

| Command | Result |
|---------|--------|
| `pip install -r requirements.txt` | **PASS** |
| `mkdocs build --strict` | **PASS** (~21s). INFO warnings for broken links to external repo paths (`../../src/...`) — non-fatal |

Preview locally:

```bash
mkdocs serve
# http://127.0.0.1:8000
```

## Restore / build / test / lint / run

| Task | Command | Verified |
|------|---------|----------|
| Restore | `pip install -r requirements.txt` | **PASS** |
| Build | `mkdocs build --strict` | **PASS** |
| Unit tests | None | — |
| Integration tests | None | — |
| Frontend tests | None | — |
| Lint | MkDocs `--strict` link check | **PASS** (with INFO link warnings) |
| Run locally | `mkdocs serve` | Not run (safe; starts local server on 8000) |

## Required services / databases / mocks

None for documentation build.

## Safe commands

- `pip install -r requirements.txt`
- `mkdocs build --strict`
- `mkdocs serve` (local preview only)

## Unsafe commands

| Command | Risk |
|---------|------|
| GitHub Pages deploy workflow | Publishes to public Pages (intended on `main` push) |

## CI reference

`.github/workflows/pages.yml`:

```bash
pip install -r requirements.txt
mkdocs build --strict
touch site/.nojekyll
# deploy-pages
```
