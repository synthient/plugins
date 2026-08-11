# Synthient for Codex

Detect residential proxies, VPNs, Tor nodes, and the proxy botnets behind them — from inside your editor.

This plugin bundles the [Synthient MCP server](https://docs.synthient.com/cli#mcp-server) and adds what it does not carry: how to read a lookup, how to integrate one, and how to migrate off another vendor.

## Install

Add this repository as a marketplace, then install the plugin:

```bash
codex plugin marketplace add /path/to/plugins
```

Then open `/plugins` in Codex and install **Synthient** from the Synthient marketplace. Start a new session afterwards — bundled skills and MCP servers are picked up at session start.

You need the CLI on your `PATH`, since the MCP server ships inside it:

```bash
brew install synthient/tap/synthient
synthient auth
```

Get a key from the dashboard at [synthient.com](https://synthient.com). The MCP server resolves credentials through the CLI's own chain: `SYNTHIENT_API_KEY` (forwarded via `env_vars`), then `.env` in the working directory, then the OS keychain. The keychain is the reliable path, since an MCP server's working directory is wherever Codex launched it.

Run `$synthient-doctor` to confirm everything resolved.

## Skills

| | |
| --- | --- |
| `$synthient` | API reference — auth, response shape, field vocabularies, credits, rate limits, errors |
| `$synthient-integrate` | Wire Synthient into this codebase, with fail-open handling and the benign-automation guard |
| `$synthient-migrate` | Move off Spur, IPQualityScore, or any other vendor — audit, map, report, rewrite, verify |
| `$synthient-triage` | Investigate an IP, a list, or a domain and get a reasoned verdict |
| `$synthient-feeds` | Parquet snapshots, filtered stream capture, CSV export, local analysis |
| `$synthient-doctor` | Diagnose credentials, scopes, credits, and MCP connectivity |

Codex also loads these on its own when a task matches the description — you do not have to invoke them by name.

## Migrating from another vendor

```
$synthient-migrate spur
```

Also takes `ipqs`, or any other vendor name. It audits every call site, maps the fields, and **shows you the report before changing anything**.

The report leads with the semantic changes, because those are what break quietly. IPQS `fraud_score` and Synthient `risk_score` are both 0–100 but calibrated differently, so a ported threshold silently shifts who gets blocked. Spur's `client.count` becomes a count of distinct device signatures, not concurrent clients. Spur operator names become Service Tags, so string comparisons stop matching — and a detector that stops matching returns clean for everything.

Vendors without a published guide go through a derived mapping, clearly marked as inferred.

## What this adds over the MCP server alone

The MCP tools return data. They carry no judgment, and several capabilities have no tool at all.

The plugin supplies the judgment — that nine of the `intelligence.categories` values are benign automation where blocking takes out Googlebot and Stripe webhooks; that `network.type` is a network classification and not a proxy verdict; that `risk_score` is a summary rather than a verdict; that Helios honeypot timestamps are milliseconds while everything else is seconds; that an empty `providers[]` is not a clean bill of health; that rate limits are per team, so a second key does not double throughput.

And it reaches what the tools cannot: Parquet download with checksum verification, filtered and long-running stream capture, CSV export, and `status` / `scopes` diagnostics.

Reference detail is fetched live from [docs.synthient.com](https://docs.synthient.com) rather than vendored, so it does not go stale.

## Links

- [Documentation](https://docs.synthient.com)
- [CLI](https://github.com/synthient/cli)
- [Go SDK](https://github.com/synthient/go-synthient)
- [contact@synthient.com](mailto:contact@synthient.com)
