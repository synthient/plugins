# Spur → Synthient: what the field table does not tell you

The name-to-name mapping lives at `https://docs.synthient.com/migration/spur.md`. Fetch it. This file covers only the places where the mapping is correct but the **meaning** shifts — the changes that compile clean and alter behavior.

## `tunnels[].operator` → `providers[].provider` — the values are not the same strings

Spur returns operator names as prose. Synthient returns **Service Tags**: stable uppercase identifiers such as `BRIGHTDATA`, `LUNAPROXY`, `NORDVPN`, `TOR`.

Any code that compares an operator against a literal, or keys a map or config block by operator name, **must be re-keyed**. A string comparison that silently stops matching produces a detector that returns clean for everything — the worst possible failure mode, because nothing errors.

Get the tag list from `https://docs.synthient.com/service-tags.md`. The public list is 273 tags and is described as a floor, not a ceiling: some tags are TLP:AMBER+STRICT and shared only under MNDA. Do not build logic that assumes the public list is complete.

Audit checklist: allowlists and denylists keyed by operator, per-provider risk weights, provider-specific branches, analytics dimensions, dashboard filters, alert rules.

## `client.count` → `intelligence.devices.length` — different unit

Spur's `count` is a count of clients observed behind the IP. Synthient's `devices[]` is a list of **distinct device signatures** derived from device clustering on torrenting and browsing traffic.

These are correlated but not interchangeable. A threshold like `count > 5` has no defensible translation. **Do not port the constant.** Keep the old value marked as needing calibration and recalibrate against real traffic — shadow mode is the right tool.

Synthient's `devices[]` also carries `os`, `version`, and `last_seen`, which Spur's scalar did not. If the old logic wanted "many clients means shared or proxied," the OS mix across `devices[]` is a stronger signal than the count alone.

## `infrastructure` → `network.type` — classification, not verdict

Spur's `infrastructure` was routinely used as a proxy verdict: `DATACENTER` meant block.

Synthient's `network.type` is explicitly **a network classification, not a proxy verdict**. `DATACENTER` covers legitimate cloud-hosted traffic — your own monitoring, partner integrations, mobile app backends.

Move the verdict to `intelligence.categories` and `intelligence.providers[]`. `DATACENTER_PROXY` is a category and means what the old `infrastructure == DATACENTER` check was reaching for. `network.type` becomes context, not a decision input.

This is the single most common Spur migration bug. If the audit finds a branch keyed on `infrastructure`, treat it as a semantic change and say so in the report.

Synthient's `network.type` also has values Spur did not: `MOBILE`, `SATELLITE`, `IN_FLIGHT_WIFI`, `CORPORATE`, `ACADEMIC`, `GOVERNMENT`. A ported switch statement will hit them. Make sure there is a default branch that fails open.

## `risks` → `intelligence.behavior` and `services` → `intelligence.categories`

Spur's `risks` and `services` split roughly along Synthient's `behavior` / `categories` line, but not cleanly, and both Synthient fields are **open sets** that grow. Never write an exhaustive switch without a default.

`intelligence.categories` includes nine benign automation values — `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, `PAYMENT_PROCESSOR`. Logic that treated a non-empty `services` array as suspicious will start blocking Googlebot and Stripe. Add the benign guard during the migration, not after.

## `intelligence.risk_score` is new

Spur had no equivalent single score. Adopting it is optional, and Synthient's own guidance is that enterprises should score internally — a 0–100 number collapses signals you may want to weigh separately. Do not restructure working per-signal logic around the score just because it is now available.

## Plumbing

| | Spur | Synthient |
| --- | --- | --- |
| Auth header | `Token: $SPUR_TOKEN` | `x-api-key: $SYNTHIENT_API_KEY` |
| Endpoint | `GET https://api.spur.us/v2/context/:ip` | `GET https://api.synthient.com/api/v4/lookup/ip/{ip}` |
| Env var | `SPUR_TOKEN` | `SYNTHIENT_API_KEY` |
| Batch | — | `POST /api/v4/lookup/ips`, up to 1,000, `ceil(n*0.9)` credits |
| Rate limit | — | 100 req/sec sustained, per **team** not per key |

New surfaces with no Spur analogue: domain lookup (`/lookup/domain/{domain}`, Helios honeypot intelligence), Parquet feed exports, real-time NDJSON streams, and gRPC. Mention them in the report as available, but do not pull them into the migration unless the user asks.

## Timestamps

Synthient's `last_seen` fields are Unix **seconds** on lookups. If the migration later touches Helios honeypot streams, those are Unix **milliseconds**. Do not share a parsing helper across both without a unit parameter.
