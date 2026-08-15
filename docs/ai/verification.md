# MITF Wallet Public Docs — Cloud Agent Verification

Last verified: 2026-08-15  
Repository: [mitf-dev-space/mitf_wallet_public_docs](https://github.com/mitf-dev-space/mitf_wallet_public_docs)  
Default branch: `main`

## Repository type

Documentation-only (MkDocs). No application source, unit tests, or required build step for Cloud Agent verification.

## Branches

| Branch | Role |
|--------|------|
| `main` | Default; GitHub Pages deploy target |
| `chore/cursor-cloud-agent-env` | Cloud Agent environment + this verification doc |

## Manifests

| Artifact | Path |
|----------|------|
| `requirements.txt` | Python/MkDocs dependencies |
| `mkdocs.yml` | Site configuration |
| CI | `.github/workflows/pages.yml` |

## Verification

| Task | Status |
|------|--------|
| Restore / build | Not required for docs-only agent work |
| Unit / integration tests | None |
| Required services | None |

## Safe commands

- Review and edit markdown under `docs/`
- `git status`, `git diff`

## Unsafe commands

| Command | Risk |
|---------|------|
| GitHub Pages deploy workflow | Publishes to public Pages |

## Cloud Agent flow

```text
clone -> review/edit docs -> commit
```

## Blockers

None for documentation-only agent tasks.
