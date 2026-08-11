---
name: synthient-integrate
description: Wire Synthient IP intelligence into a codebase — pick the integration point, choose per-request lookups or a streamed proxy cache, and implement it with fail-open error handling, caching, and the benign-automation guard. Use when adding proxy, VPN, or bot detection to an application.
---

# Integrate Synthient

Read the field vocabularies and invariants from the `synthient` skill before writing code.

## 1. Find the integration point

If the user named one, use it. Otherwise read the codebase and propose one — do not guess silently.

The usual points, in rough order of value: signup and registration, login and authentication, checkout and payment, content submission, API gateway or middleware, rate limiter.

Confirm the choice before writing code. Where the check goes determines what the failure modes cost, and that is the user's call.

## 2. Choose the pattern

**Per-request lookup.** One `GET /api/v4/lookup/ip/{ip}` per event, cached. Right for signup, login, checkout — anywhere the event rate is well under the lookup budget. Costs 1 credit per uncached address.

**Streamed proxy cache.** Consume `GET /api/v4/feeds/proxies/stream` into an in-memory set with a 5–10 minute TTL, then test membership at request time. Zero per-request latency, zero credits, and this is the canonical residential-proxy pattern — Synthient's own risk guidance recommends it, because residential proxies have short lifespans and a recent-window cache both catches them and limits false positives. Right for high-volume traffic and edge filtering. Needs the `PROXY_FIREHOSE` scope and a long-lived connection.

**Both.** Cache for the hot path, lookup for enrichment on the events that matter. This is what most production integrations end up doing.

Pick based on event volume and whether a lookup can sit in the request path. Say which you picked and why.

## 3. Choose the transport

**Go** — use the SDK: `go get -u github.com/synthient/go-synthient/v2` (Go 1.25+). `synthient.NewClient(os.Getenv("SYNTHIENT_API_KEY"))`, then `GetIP`, `GetIPs`, `GetDomain`, `GetAccount`. Streams are `iter.Seq2`. Every method takes a trailing `*synthient.RequestOptions` that may be `nil`.

**Everything else** — plain HTTP. Python, Node, Java, and Ruby SDKs are in development and **do not exist**. Never import one. Fetch `https://docs.synthient.com/ipapi.md` for the exact request and response shapes.

**gRPC** — available at `grpc.synthient.com:443` with schema over reflection, if the codebase is already gRPC-native.

## 4. Implement

Non-negotiables, in the order they get forgotten:

**Fail open.** A 402, 429, 500, or 503 must never block a real user. A billing state or a Synthient outage locking out your signup flow is a worse outcome than missing a proxy. Wrap the lookup so any error path yields "no signal" and the request proceeds. Make this explicit in the code, not incidental.

**Never block on benign automation.** `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, `PAYMENT_PROCESSOR`. Write this guard first, before the blocking logic, so it cannot be forgotten later.

**Never block on `network.type` alone.** It is a network classification, not a proxy verdict. `DATACENTER` includes your own monitoring and your partners' integrations. The verdict lives in `intelligence.categories` and `intelligence.providers[]`.

**Server-side only.** The API key never reaches a browser or a mobile client. Read it from the environment; never commit it. One key per environment so they rotate independently.

**Timeout and context.** A lookup in the request path needs a deadline — a few hundred milliseconds — and a context that cancels with the request.

**Cache.** 5–10 minutes per address. Cuts credits and latency together. Residential proxy assignments change fast enough that a longer TTL loses accuracy.

**Batch where you can.** `POST /api/v4/lookup/ips` takes up to 1,000 addresses at `ceil(n*0.9)` credits. Never loop the single endpoint over a list.

**Default branches everywhere.** `categories`, `behavior`, and `providers[].type` are open sets that grow. An unrecognized value must not fail closed.

**Timestamps.** `last_seen` is Unix seconds on lookups. Helios honeypot streams are Unix **milliseconds**. Do not share a parser.

## 5. Decide the policy with the user

Do not invent thresholds. Ask what should happen on a hit — block, step up to MFA, flag for review, log only — and start conservative. Logging first, enforcing once the data is understood, is almost always right. Point at `https://docs.synthient.com/guides/risk.md`: `risk_score` is a summary, not a verdict, and per-signal logic beats a single threshold.

Grounding from that guide, if the user wants a starting point: ISP and datacenter proxies are used almost exclusively for automation and can be treated as high risk. VPNs are more ambiguous — medium to high. Residential proxies need the recent-window cache and corroborating signals, because real users sit behind them too.

## 6. Verify

Run the `synthient-doctor` skill for credentials, scopes, and credits. Do a live lookup against a known address and confirm the parsed result end to end. Test the failure path explicitly — an unreachable API must let a request through, and that deserves a test rather than a manual check.
