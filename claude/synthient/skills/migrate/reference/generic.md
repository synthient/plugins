# Migrating from a vendor with no published guide

Synthient publishes migration guides for Spur and IPQualityScore only. For MaxMind, ipinfo, ipdata, Castle, Fingerprint, IP2Location, AbuseIPDB, or anything else, derive the mapping.

**Say so.** Every mapping produced this way is inferred from the vendor's response shape, not read from an authoritative table. Mark the whole report as inferred and recommend shadow mode before cutover. Do not present a derived mapping with the same confidence as the Spur or IPQS tables.

## Derive the mapping

Work from what the codebase **consumes**, not from the vendor's full response. The audit already produced that list. A vendor may return sixty fields while the codebase reads four.

For each consumed field:

1. Find the vendor's documented meaning: its own docs, or the type definitions and parsing code in the repo. Do not guess from the field name; `risk`, `score`, and `threat` mean different things at every vendor.
2. Find the Synthient field with the same **meaning**, using `https://docs.synthient.com/ipapi.md` for the target shape.
3. Classify it: direct, semantic change, or no equivalent.
4. Find every piece of decision logic that reads it. That is where a semantic change does its damage.

## The four traps

These recur at essentially every vendor. Check each one explicitly.

**Score calibration.** Any 0–100 or 0–1 score maps to `intelligence.risk_score` in *range* but never in *calibration*. Thresholds do not transfer. Never port the constant; mark it for recalibration and recommend shadow mode.

**Booleans to membership tests.** Synthient has no `is_proxy` / `is_vpn` / `is_tor` fields. Every vendor boolean becomes a membership test over `intelligence.categories` or `intelligence.providers[].type`, and those are open sets that need a default branch. Synthient also adds `last_seen` recency, which a boolean cannot carry.

**Connection type as a verdict.** Most vendors return something like `connection_type` or `usage_type` that teams use as a block signal. Synthient's `network.type` is explicitly a network classification, not a proxy verdict. Move the verdict to `categories` and `providers[]`.

**Benign automation.** Most vendors lump crawlers and bots in with abuse. Synthient separates them: `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, `PAYMENT_PROCESSOR` are benign, and blocking them is a bug. Any ported bot check needs the benign guard added.

## Also check

- **Provider or operator names.** If the vendor named proxy and VPN operators, Synthient uses Service Tags, stable uppercase identifiers. Re-key every string comparison against `https://docs.synthient.com/service-tags.md`. A comparison that stops matching fails silently and returns clean for everything.
- **Auth.** Header `x-api-key`, never a URL path or query parameter. If the old key travelled in a URL it is in access logs; rotate it before decommissioning.
- **Errors.** Synthient uses HTTP status codes, not a `success` field in a 200 body. 402 (credits exhausted) is new and should fail open.
- **Batching.** `POST /api/v4/lookup/ips` takes up to 1,000 addresses at a 10% discount. Most integrations being migrated loop.
- **Rate limits.** Per team, not per key. A second key does not double throughput.
- **Geolocation precision.** Vendors differ on how they resolve city and coordinates. If the codebase makes decisions on distance or location match, expect different results even where the field names align.
- **Fields with no equivalent.** Common gaps: browser and user-agent parsing, email or phone reputation, device fingerprinting, historical abuse counters. These need the logic rethought, not remapped. List them prominently, because a migration that quietly drops a signal is worse than one that names the loss.

## What Synthient adds

Worth mentioning in the report, but do not scope-creep the migration:

- Domain lookup backed by Helios honeypot sensors
- Parquet feed exports and real-time NDJSON streams: the live proxy cache pattern replaces per-request lookups at high volume
- Device clustering with per-device OS, version, and recency
- Provider attribution with `last_seen`
- gRPC alongside HTTP
