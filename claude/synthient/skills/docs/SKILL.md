---
name: docs
description: Synthient API reference — authentication, the IP and domain lookup response shape, field vocabularies, credit costs, rate limits, feeds and Helios, and the errors that matter. Use when writing or reviewing code that calls api.synthient.com, when interpreting a lookup response or a risk_score, or when any Synthient field name appears (intelligence.categories, providers, network.type, behavior, service tags).
when_to_use: Triggers on Synthient, api.synthient.com, x-api-key, residential proxy detection, VPN or Tor detection, proxy botnets, risk_score, intelligence.categories, service tags, Helios, firehose, feed snapshots, go-synthient.
---

# Synthient API reference

Synthient detects anonymized network traffic: residential proxies, VPNs, Tor nodes, private relays, and the proxy botnets behind them.

**Prefer the MCP tools over hand-rolled HTTP.** If this plugin's server is connected you have `lookup_ip`, `lookup_domain`, `get_account`, `list_feed_streams`, `list_feed_snapshots`, `feed_snapshot_meta`, `sample_stream`, and `grpc_schema`. Use them for data. Write HTTP only when producing code that ships.

## Invariants

These are load-bearing. Do not contradict them, and do not go fetch a page to confirm one.

**Authentication.** Every surface takes one API key — a UUID — in the `x-api-key` header. Never `Authorization: Bearer`. Never a query parameter. **Never from a browser**: all surfaces are server-to-server only. The conventional environment variable is `SYNTHIENT_API_KEY`.

**Endpoints.** HTTP is `https://api.synthient.com`, everything under `/api/v4`. gRPC is `grpc.synthient.com:443` (TLS required), service `synthient.v1.SynthientService`, schema over server reflection. The `/api/v4` vs `synthient.v1` version mismatch is intentional — do not "fix" it. Never use `api.synthient.com` or port `50051` for gRPC.

**SDKs.** Only Go has one: `github.com/synthient/go-synthient/v2`, Go 1.25+. Python, Node, Java, and Ruby are in development. In those languages **write plain HTTP** — never import an SDK that does not exist.

**Credits.** 1 per IP lookup. `ceil(n * 0.9)` per batch, max 1,000 addresses, duplicates and invalid entries stripped before billing. 1 per domain lookup. Feeds, streams, and `/account/me` are free. Batching is always cheaper than looping.

**Nine categories are benign automation.** `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, `PAYMENT_PROCESSOR`. **Blocking these is usually a bug** — it takes out Googlebot, Stripe webhooks, and your own uptime checks.

**The enums are open sets.** `intelligence.categories`, `intelligence.behavior`, and `providers[].type` grow over time. Never write an exhaustive switch without a default branch, and never fail closed on an unrecognized value.

**Timestamp units differ.** Unix **seconds** for lookups, feed metadata, and the `proxies` / `anonymizers` / `torrents` streams. Unix **milliseconds** for all four Helios honeypot streams. gRPC uses `google.protobuf.Timestamp` throughout. This is the single most common integration bug.

**An empty `providers[]` is not a clean bill of health.** It means no provider attribution, not "not a proxy."

**`network.type` is a network classification, not a proxy verdict.** `DATACENTER` alone is not grounds to block. The verdict lives in `intelligence.categories` and `intelligence.providers[]`.

**`risk_score` is a summary, not a verdict.** 0–100, derived from device clustering, proxy/VPN signals, and honeypot data. Synthient's own guidance is that enterprises should score internally, because a single number collapses signals you may want to weigh differently.

**Rate limits are per team, not per key.** Issuing a second key does not double throughput. Lookups sustain 100 req/sec (burst 200); `/account/me` is 10 req/sec. Responses carry `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset`.

## Response shape

`GET /api/v4/lookup/ip/{ip}` returns three objects beside `ip`:

- `network` — `asn`, `isp`, `type`, `org`, `domain`, `abuse_email`, `abuse_phone`
- `location` — `country`, `state`, `city`, `timezone`, `latitude`, `longitude`, `geo_hash`
- `intelligence` — `risk_score`, `behavior[]`, `categories[]`, `devices[{os, version, last_seen}]`, `providers[{provider, type, last_seen}]`

`network.type` is a closed-ish set of 8: `MOBILE`, `SATELLITE`, `IN_FLIGHT_WIFI`, `RESIDENTIAL`, `CORPORATE`, `ACADEMIC`, `DATACENTER`, `GOVERNMENT`.

`intelligence.behavior` covers a 90-day window: `PROGRAMMATIC_TRAFFIC`, `ACTIVE_CRAWLER`, `TORRENTING`, `TOR_USER`, `CREDENTIAL_STUFFING`, `COMPROMISED_DEVICE`, `MALICIOUS_TRAFFIC`.

The anonymization categories, as distinct from the nine benign ones: `FREE_VPN`, `COMMERCIAL_VPN`, `ENTERPRISE_VPN`, `MOBILE_PROXY`, `BLOCKCHAIN_PROXY`, `RESIDENTIAL_PROXY`, `PUBLIC_PROXY`, `DATACENTER_PROXY`, `TOR_NODE`, `PRIVATE_RELAY`, `BOTNET`.

`providers[].provider` is a **Service Tag**: a stable uppercase identifier like `BRIGHTDATA`, `LUNAPROXY`, `NORDVPN`, `TOR`, `APPLE`. The public list is 273 tags and is a floor, not a ceiling — match against it, do not assume it is exhaustive.

## Errors

| Status | Meaning |
| --- | --- |
| 400 | Malformed IP, bad cursor, oversized batch. Fix it; do not retry |
| 401 | Missing or invalid `x-api-key` |
| 402 | Lookup credits exhausted. Streams and exports still work |
| 403 | Key is valid but lacks the scope |
| 404 | Snapshot or domain genuinely absent |
| 429 | Rate limit or concurrent-stream limit |
| 500 / 503 | Retry with backoff and jitter |

Bodies are `{"detail": "..."}` or `{"title": "Validation error", "errors": {"field": ["message"]}}`. An empty body is transient — retry. Backoff recipe: start at 1s, double, cap at 60s, ±25% jitter, give up after 5–8 attempts, honor `Retry-After`.

## Fetch detail on demand

Fetch **one page**, not the whole blob. Each route below serves clean raw markdown.

| Topic | URL |
| --- | --- |
| IP + domain lookup, full field reference | `https://docs.synthient.com/ipapi.md` |
| Account, scopes, quota | `https://docs.synthient.com/account.md` |
| API keys, scope model, rotation | `https://docs.synthient.com/authentication.md` |
| Status codes, error bodies, retries | `https://docs.synthient.com/errors.md` |
| Per-endpoint limits and headers | `https://docs.synthient.com/rate-limits.md` |
| gRPC methods and status mapping | `https://docs.synthient.com/grpc.md` |
| Go SDK | `https://docs.synthient.com/sdk.md` |
| CLI commands and flags | `https://docs.synthient.com/cli.md` |
| Risk scoring: derivation and caveats | `https://docs.synthient.com/guides/risk.md` |
| The 273 service tags | `https://docs.synthient.com/service-tags.md` |
| Parquet snapshots | `https://docs.synthient.com/enterprise/feeds.md` |
| Real-time NDJSON streams | `https://docs.synthient.com/enterprise/firehose.md` |
| Helios honeypot platform | `https://docs.synthient.com/enterprise/helios.md` |
| Data sourcing and IPv6 coverage | `https://docs.synthient.com/methodology.md` |
| Migrating from Spur | `https://docs.synthient.com/migration/spur.md` |
| Migrating from IPQualityScore | `https://docs.synthient.com/migration/ipqs.md` |
| Everything at once (~14k tokens, last resort) | `https://docs.synthient.com/llms-full.txt` |

## Related skills

`/synthient:integrate` to wire Synthient into this codebase. `/synthient:triage` to investigate an address. `/synthient:migrate` to move off another vendor. `/synthient:feeds` for bulk data. `/synthient:doctor` when something is not working.
