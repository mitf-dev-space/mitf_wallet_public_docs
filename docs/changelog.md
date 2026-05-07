# Changelog & releases

## This documentation site

- **[GitHub Releases — mitf_wallet_public_docs](https://github.com/anstwechy/mitf_wallet_public_docs/releases)** — release notes and tags for this repo.
- **Feeds:** `/feed_rss_updated.xml` and `/feed_json_updated.json` at the site root for Slack/intranet watchers.
- **Source:** [Repository on GitHub](https://github.com/anstwechy/mitf_wallet_public_docs).

!!! note "Platform and API binaries"
    Behavioral and contract changes to wallet services are shipped from the **application repositories**, not this docs repo.

### Suggested release notes content

When tagging a docs release, mention:

- New or renamed REST paths, headers, or gRPC RPCs.
- Breaking changes to idempotency, async polling, or error shapes.
- New operational requirements (config keys, probes, migrations).

See **API versioning** in [API reference](reference/api.md#api-versioning) for how we describe API evolution.
