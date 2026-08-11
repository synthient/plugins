# IPQualityScore → Synthient: what the field table does not tell you

The name-to-name mapping lives at `https://docs.synthient.com/migration/ipqs.md`. Fetch it. This file covers only where the mapping is correct but the **meaning** shifts.

## `fraud_score` → `risk_score` — same range, different calibration

Both are 0–100. They are **not the same scale**. IPQS derives `fraud_score` from its own model; Synthient derives `risk_score` from device clustering, proxy and VPN signals, and honeypot data.

**Every ported threshold is wrong.** `fraud_score > 75` does not become `risk_score > 75`. There is no conversion factor, and inventing one is worse than admitting the gap.

What to do: keep the old constant, mark it clearly as needing calibration, and recommend shadow mode — run both, log the disagreements, pick the new threshold from real traffic. If the user wants a number today, say plainly that any number given now is a guess.

Synthient's own guidance goes further: treat the score as a summary, not a verdict, and score internally where you can. A migration is a good moment to move decision logic onto the underlying signals — `categories`, `providers[]`, `behavior` — with the score as one input rather than the whole decision. Raise it; do not force it.

## `proxy` / `vpn` / `tor` booleans → membership tests

IPQS returns flat booleans. Synthient has no boolean fields. The equivalents are membership tests:

| IPQS | Synthient |
| --- | --- |
| `proxy` | `intelligence.categories` contains any of `RESIDENTIAL_PROXY`, `DATACENTER_PROXY`, `PUBLIC_PROXY`, `MOBILE_PROXY`, `BLOCKCHAIN_PROXY` |
| `vpn` | `intelligence.categories` contains any of `FREE_VPN`, `COMMERCIAL_VPN`, `ENTERPRISE_VPN` |
| `tor` | `intelligence.categories` contains `TOR_NODE`, or `providers[].provider == "TOR"` |

Two things change. First, these are **open sets** — new proxy and VPN categories get added, so a hardcoded list needs a default branch and periodic review. Second, IPQS's booleans are timeless while Synthient's `providers[].last_seen` tells you *when* the IP was last confirmed on that provider. A proxy last seen ninety days ago is a much weaker signal than one seen this morning, and residential proxies in particular have short lifespans. Recency is available; use it.

`active_vpn` and `active_tor` map onto `last_seen` recency rather than onto separate fields.

## `is_crawler` / `bot_status` / `recent_abuse` → `intelligence.behavior` — read this before porting

The nearest values are `ACTIVE_CRAWLER`, `PROGRAMMATIC_TRAFFIC`, and `MALICIOUS_TRAFFIC` / `CREDENTIAL_STUFFING`.

**A naive port starts blocking Googlebot and Stripe webhooks.** IPQS's `is_crawler` and `bot_status` conflate "automated" with "unwanted." Synthient separates them: `intelligence.categories` carries nine benign automation values — `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, `PAYMENT_PROCESSOR` — where blocking is a bug.

So `is_crawler == true → block` must become: check `behavior` for the abuse signal **and** check that `categories` carries no benign automation value. Never a bare behavior check.

`recent_abuse` maps to `MALICIOUS_TRAFFIC`, `CREDENTIAL_STUFFING`, or `COMPROMISED_DEVICE` depending on what the old logic meant by abuse. Ask the user which, rather than guessing — the three lead to different responses.

`intelligence.behavior` covers a rolling **90-day window**. If IPQS's `recent_abuse` used a different window, the rate of positives changes even with identical logic.

## `connection_type` → `network.type` — classification, not verdict

Same trap as everywhere else: `network.type` is a network classification. `DATACENTER` is not grounds to block on its own — the verdict belongs in `categories` and `providers[]`. Synthient adds `SATELLITE`, `IN_FLIGHT_WIFI`, `ACADEMIC`, and `GOVERNMENT`, which a ported switch will not handle. Default branch, fail open.

## `mobile` / `operating_system` / `browser` / `device_model` → `devices[]`

IPQS returns scalars describing one inferred device. Synthient returns an **array** of distinct device signatures, each with `os`, `version`, and `last_seen`.

`mobile == true` becomes: any `devices[].os` in `ANDROID`, `IOS`. But note that a multi-entry `devices[]` on a single IP is itself a signal — device clustering is one of the inputs to the risk score. Code that collapsed the array to its first element throws away the more useful signal.

Browser information has no direct Synthient equivalent. Flag it as **no equivalent** in the report; if the old logic depended on it, that logic needs rethinking rather than remapping.

## Plumbing

| | IPQS | Synthient |
| --- | --- | --- |
| Auth | Key **in the URL path**: `/api/json/ip/$KEY/$IP` | `x-api-key` header. Never in the URL |
| Endpoint | `GET https://www.ipqualityscore.com/api/json/ip/:key/:ip` | `GET https://api.synthient.com/api/v4/lookup/ip/{ip}` |
| Env var | `IPQS_KEY` | `SYNTHIENT_API_KEY` |
| Batch | — | `POST /api/v4/lookup/ips`, up to 1,000, `ceil(n*0.9)` credits |
| Rate limit | — | 100 req/sec sustained, per **team** not per key |

The key moving out of the URL is a security improvement worth calling out: IPQS keys leak into access logs, proxy logs, and browser history. During migration, check whether the old key ended up anywhere it should be scrubbed, and rotate it before decommissioning.

New surfaces with no IPQS analogue: domain lookup backed by Helios honeypot data, Parquet feed exports, real-time NDJSON streams, gRPC.

## Error handling

IPQS signals failure with a `success: false` field in a 200 response. Synthient uses **HTTP status codes**: 400 malformed, 401 bad key, 402 credits exhausted, 403 missing scope, 404 absent, 429 rate limited, 500/503 transient.

Any error handling that checks a body field instead of the status code needs rewriting, not remapping. And 402 is new — credits can run out. Decide explicitly whether that fails open or closed. It should fail open: a billing state must never lock out real users.
