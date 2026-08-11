---
name: synthient-feeds
description: Work with Synthient bulk data. Download and checksum-verify Parquet snapshots, capture filtered or long-running NDJSON streams, export CSV, and analyze the result locally with the synthient CLI. Use for downloading a proxy feed, capturing the firehose, building a blocklist, or analyzing torrent and honeypot data.
---

# Synthient feeds and streams

## Use the right surface

| Task | Surface |
| --- | --- |
| List streams, list snapshots, snapshot metadata and checksum | **MCP tools**: `list_feed_streams`, `list_feed_snapshots`, `feed_snapshot_meta` |
| A quick look at live events, up to 500, up to 120s | **MCP tool**: `sample_stream` |
| Downloading actual Parquet bytes | **CLI**: no MCP tool exists |
| Filtered capture, long runs, reconnect, output to file | **CLI**: `sample_stream` has no filters and hard caps |
| CSV output | **CLI**: MCP has no CSV path |

Discover with the MCP tools; move the bytes with the CLI. Do not shell out for something a tool already answers.

The CLI needs to be installed: `brew install synthient/tap/synthient`, or `go install github.com/synthient/cli/cmd/synthient@latest`. It uses its own credential chain: `SYNTHIENT_API_KEY`, then `.env` in the working directory, then the OS keychain via `synthient auth`. Run the `synthient-doctor` skill if anything fails.

## Streams

Seven identifiers, shared by exports and live streams:

| Stream | Aliases | Live? |
| --- | --- | --- |
| `proxies` | `proxy` | yes |
| `anonymizers` | `anonymizer` | yes |
| `torrents` | `torrent` | yes |
| `honeypot_http` | `helios_http`, `http` | yes |
| `honeypot_https` | `helios_https`, `https`, `tls` | yes |
| `honeypot_dns` | `helios_dns`, `dns` | **not via CLI or MCP** |
| `honeypot_adb` | `helios_adb`, `adb` | **not via CLI or MCP** |

`honeypot_dns` and `honeypot_adb` have `_STREAM` scopes and the API serves them, but the Go SDK's `StreamHeliosDNS` and `StreamHeliosADB` are commented out, so `synthient stream` and `sample_stream` both reject them with "real-time stream not supported". Use snapshots for those two, or call the stream endpoint directly.

Each is gated by its own scope: `PROXY_FEEDS`, `ANONYMIZERS_FEED`, `HONEYPOT_HTTP_STREAM`, and so on, with `PROXY_FIREHOSE` for the proxy stream. Check with `synthient scopes --format json`. Scopes are granted by Synthient, not self-served.

## Snapshots

Snapshot ids: `latest` (most recent hourly), `YYYY-MM-DD` (daily rollup), or `YYYY-MM-DD/HH` (a specific hour, **current UTC day only**). At 00:30 UTC the previous day's hourlies roll up into the daily and the per-hour artifacts are deleted, so an hourly from two days ago does not exist, and that is expected, not an error.

```bash
synthient download proxies latest ./proxies.parquet --verify
synthient feeds download proxies 2026-08-09 ./proxies-0809.parquet --verify
```

Always pass `--verify`. It checks the SHA-256 from the snapshot metadata against the downloaded bytes. The destination must end in `.parquet`. Presigned URLs expire in 24 hours and are minted per request, so never cache or share one.

Feeds cost no lookup credits. Snapshots are large: the daily `proxies` rollup runs to hundreds of megabytes and tens of millions of rows, so check `feed_snapshot_meta` for `size_bytes` and `row_count` before downloading.

## Live capture

```bash
synthient stream proxies --output proxies.ndjson --duration 1h --reconnect
synthient stream proxies --filter type=RESIDENTIAL_PROXY --max-events 5000 --output res.ndjson
```

`stream` is always NDJSON and has no `--format`. `--pretty` is for eyeballing only, never for piping. `--filter field=value` repeats, ANDs, and takes dot notation for nested fields, applied client-side. Bound the run with `--max-events` or `--duration`, and use `--reconnect` for anything long.

**A clean close after roughly 30 minutes is normal.** The server cycles healthy connections. Reconnect immediately; treating it as an error and backing off just loses data. Blank-line heartbeats arrive every 15 seconds. Stream connection establishment is rate limited to 0.5 req/sec, so reconnect promptly but do not hammer.

Helios honeypot streams carry a `meta` block (`pool_id`, `provider`, `proxy_ip`, `server`) attributing attacker traffic to the proxy exit it came through.

**Timestamps in the Helios streams are Unix milliseconds.** The `proxies`, `anonymizers`, and `torrents` streams are Unix seconds. Do not share a parser across the two without a unit parameter.

## Analyze

Query Parquet in place; never load a snapshot into memory to inspect it:

```bash
duckdb -c "SELECT count(*) FROM './proxies.parquet'"
duckdb -c "SELECT provider, count(*) FROM './proxies.parquet' GROUP BY 1 ORDER BY 2 DESC"
```

The `proxies` schema is `asn` (uinteger), `country_code`, `ip`, `provider`, `timestamp` (bigint, Unix seconds), `type`. Call `feed_snapshot_meta` for any other stream's columns before you download, which is usually enough to write the query up front.

For NDJSON captures, `jq` over the file. For spreadsheets, the finite read commands take `--format csv --output file.csv`; `stream` does not.

## The blocklist pattern

For a live proxy blocklist, do not download snapshots on a cron. Consume `synthient stream proxies` into a set with a 5–10 minute TTL and test membership at request time. Residential proxies have short lifespans, so a recent window both catches them and limits false positives. Snapshots are for analysis and backfill; the stream is for enforcement.
