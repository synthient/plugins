# Synthient AI Plugins

Plugins that bring [Synthient](https://synthient.com) IP intelligence into AI coding agents — detecting residential proxies, VPNs, Tor nodes, and the proxy botnets behind them.

## Claude Code

```
/plugin marketplace add synthient/plugins
/plugin install synthient@synthient
```

Bundles the Synthient MCP server and adds six skills: API reference, codebase integration, migration off another vendor, address triage, bulk feeds, and setup diagnosis. See [`claude/synthient`](./claude/synthient).

Requires the CLI on your `PATH`: `brew install synthient/tap/synthient`.

## Codex

Not yet built.

## Layout

```
.claude-plugin/marketplace.json   the "synthient" marketplace
claude/synthient/                 the Claude Code plugin
codex/                            reserved
```

## Links

- [Documentation](https://docs.synthient.com)
- [CLI](https://github.com/synthient/cli)
- [Go SDK](https://github.com/synthient/go-synthient)
- [contact@synthient.com](mailto:contact@synthient.com)
