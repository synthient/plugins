# Synthient AI Plugins

Plugins that bring [Synthient](https://synthient.com) IP intelligence into AI coding agents, detecting residential proxies, VPNs, Tor nodes, and the proxy botnets behind them.

## Claude Code

```
/plugin marketplace add synthient/plugins
/plugin install synthient@synthient
```

Bundles the Synthient MCP server and adds six skills: API reference, codebase integration, migration off another vendor, address triage, bulk feeds, and setup diagnosis. See [`claude/synthient`](./claude/synthient).

> [!NOTE]
> Requires the CLI on your `PATH`: `brew install synthient/tap/synthient`.

## Codex

```
codex plugin marketplace add synthient/plugins
codex plugin add synthient@synthient
```

Then start a new session. Bundled skills and MCP servers are picked up at session start.

Bundles the same MCP server and the same six skills, invoked as `$synthient`, `$synthient-migrate`, and so on. See [`codex/synthient`](./codex/synthient).

> [!NOTE]
> Requires the CLI on your `PATH`: `brew install synthient/tap/synthient`.

## Layout

```
.claude-plugin/marketplace.json     the Claude Code marketplace
.agents/plugins/marketplace.json    the Codex marketplace
claude/synthient/                   the Claude Code plugin
codex/synthient/                    the Codex plugin
```

Both plugins carry the same six skills and bundle the same MCP server. They differ only where the two hosts do: Claude Code namespaces skills as `/synthient:<name>` and can prompt for the API key at enable time via `userConfig`, while Codex names skills `synthient-<name>` and relies on the CLI's own credential chain.

## Links

- [Documentation](https://docs.synthient.com)
- [CLI](https://github.com/synthient/cli)
- [Go SDK](https://github.com/synthient/go-synthient)
- [contact@synthient.com](mailto:contact@synthient.com)
