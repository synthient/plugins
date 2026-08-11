---
name: doctor
description: Diagnose a Synthient setup, covering whether the CLI is installed, which credential source is active, whether the account is reachable, which scopes are granted, how many lookup credits remain, and whether the MCP tools are connected. Use when a Synthient call fails or before relying on one.
when_to_use: Triggers on a Synthient 401, 402, or 403, "synthient not working", "no api key found", the MCP server failing to start, or checking credits and scopes before a bulk job.
allowed-tools: Bash(synthient status:*) Bash(synthient scopes:*) Bash(synthient account:*) Bash(synthient --version) Bash(command -v synthient)
---

# Diagnose Synthient setup

The MCP server has no `status` or `scopes` tool, so this runs the CLI. Work down the list and stop at the first failure; each check depends on the one above it.

## 1. Is the CLI installed?

```bash
command -v synthient && synthient --version
```

Not found → `brew install synthient/tap/synthient`, or `go install github.com/synthient/cli/cmd/synthient@latest`.

This also explains an MCP server that will not start. The plugin's `.mcp.json` runs `synthient mcp`; with no binary on `PATH` it fails at launch and shows up in the `/plugin` **Errors** tab.

## 2. Are credentials resolving?

```bash
synthient status --format json
```

Reports `auth_source`, `account_reachable`, the resolved endpoints, the config path, and the active profile, without printing the key. Credential order, first match wins:

1. `SYNTHIENT_API_KEY` in the environment
2. `.env` in the working directory
3. the OS keychain, written by `synthient auth`

`auth_source: "missing"` → run `synthient auth` to store a key in the keychain, or set the plugin's **Synthient API key** option in `/plugin`, which passes it to the MCP server as an environment variable. Get a key from the dashboard at https://synthient.com.

A note for MCP specifically: the working directory of an MCP server is wherever the client launched it, so a `.env` in your project may not be visible to it. The keychain and the plugin option are the reliable paths.

The plugin's API key option is optional by design; leaving it blank passes an empty value, which the CLI treats as unset and falls through to `.env` and the keychain. Setting it does not break an existing `synthient auth` setup; it takes precedence over it.

## 3. Is the account reachable, and are there credits?

```bash
synthient account --format json
```

Or call the `get_account` MCP tool. Returns the organization, granted scopes, and `lookup_quota` with `credits` and `resets_in`.

`credits: 0` → lookups return **402**. Feeds, streams, and `/account/me` still work; they cost no credits. Fail open in application code: a billing state must never lock out real users.

## 4. Are the right scopes granted?

```bash
synthient scopes --format json
```

Shows every scope with a `granted` flag and what it permits. `BASIC` covers lookups and account. Feeds and streams each need their own: `PROXY_FEEDS`, `PROXY_FIREHOSE`, `ANONYMIZERS_FEED` / `_STREAM`, `TORRENTS_FEED` / `_STREAM`, and `HONEYPOT_{HTTP,HTTPS,DNS,ADB}_{FEED,STREAM}`.

A missing scope gives **403**. Scopes cannot be self-granted; email contact@synthient.com.

## 5. Are the MCP tools connected?

Check whether `lookup_ip`, `lookup_domain`, `get_account`, `list_feed_streams`, `list_feed_snapshots`, `feed_snapshot_meta`, `sample_stream`, and `grpc_schema` are available in this session. If not, and steps 1–2 passed, run `/reload-plugins`, then check the `/plugin` Errors tab.

The server exits at startup if it finds no API key, so a missing key looks like a broken server rather than an auth error. Step 2 is what tells them apart.

## Status codes

| Code | Cause | Fix |
| --- | --- | --- |
| 401 | Missing or invalid key | Step 2. Key is a UUID in the `x-api-key` header, never `Authorization: Bearer` |
| 402 | Credits exhausted | Step 3. Feeds and streams unaffected |
| 403 | Valid key, missing scope | Step 4. Email contact@synthient.com |
| 429 | Rate limited | Limits are per **team**, not per key, and a second key does not help. Back off with jitter, honor `Retry-After` |
| 500 / 503 | Transient | Retry: 1s, doubling, cap 60s, ±25% jitter, give up after 5–8 |

## Report

State what passed, what failed, and the one next action. If everything passes, say so with the concrete numbers (auth source, organization, credits remaining, scopes granted) rather than just "healthy."
