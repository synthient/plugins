---
name: triage
description: Investigate an IP address, a list of addresses, or a domain with Synthient and return a reasoned verdict rather than a raw score, with provider attribution, benign-automation check, behavior and device signals, and a recommended action. Use when asked whether an address is a proxy, a VPN, Tor, a bot, or safe to block.
when_to_use: Triggers on "is this IP a proxy", "check this address", "look up 1.2.3.4", "should we block this IP", "what is this domain doing", or investigating addresses from a log or abuse report.
argument-hint: [ip, list of ips, or domain]
---

# Triage an address

Use the MCP tools: `lookup_ip` for one or many addresses, `lookup_domain` for a domain. Do not hand-roll HTTP for an investigation. If the tools are unavailable, run `/synthient:doctor` first rather than working around it.

Batch multiple addresses into a single `lookup_ip` call: it costs `ceil(n*0.9)` credits instead of `n`.

Subject is `$ARGUMENTS`.

## Read the response in this order

**1. Provider attribution: `intelligence.providers[]`.** The strongest signal. Each entry has `provider` (a Service Tag like `BRIGHTDATA` or `NORDVPN`), `type`, and `last_seen`.

Weigh `last_seen` heavily. Residential proxies have short lifespans; an IP last confirmed on a proxy network this morning is a different proposition from one last seen three months ago. Say which it is.

An empty `providers[]` is **not** a clean bill of health. It means no attribution, not "not a proxy."

**2. Categories: `intelligence.categories`.** Split them before reasoning:

*Anonymization:* `RESIDENTIAL_PROXY`, `DATACENTER_PROXY`, `PUBLIC_PROXY`, `MOBILE_PROXY`, `BLOCKCHAIN_PROXY`, `FREE_VPN`, `COMMERCIAL_VPN`, `ENTERPRISE_VPN`, `TOR_NODE`, `PRIVATE_RELAY`, `BOTNET`.

*Benign automation:* `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, `PAYMENT_PROCESSOR`. **If the address carries one of these, lead with it.** Recommending a block on Googlebot or Stripe is the failure mode this skill exists to prevent.

The set is open. An unfamiliar value is not automatically hostile; say you do not recognize it rather than guessing.

**3. Behavior: `intelligence.behavior`.** A 90-day window: `PROGRAMMATIC_TRAFFIC`, `ACTIVE_CRAWLER`, `TORRENTING`, `TOR_USER`, `CREDENTIAL_STUFFING`, `COMPROMISED_DEVICE`, `MALICIOUS_TRAFFIC`.

`CREDENTIAL_STUFFING`, `MALICIOUS_TRAFFIC`, and `COMPROMISED_DEVICE` are direct abuse evidence. `ACTIVE_CRAWLER` and `PROGRAMMATIC_TRAFFIC` are automation, which is only abuse in context, so cross-check against benign automation before treating them as hostile.

**4. Devices: `intelligence.devices[]`.** Distinct device signatures with `os`, `version`, `last_seen`. Several distinct signatures on one address suggests sharing: CGNAT, corporate egress, or a proxy exit. Corroborating, not conclusive.

**5. Network: `network.type`, `asn`, `isp`, `org`.** Context only. **`network.type` is a network classification, not a proxy verdict.** Never recommend a block on `DATACENTER` alone.

**6. Risk score: `intelligence.risk_score`.** Report it, but treat it as a summary. It is built from device clustering, proxy signals, and honeypot data, and Synthient's own guidance is to score internally rather than lean on the number. Do not lead with it, and never present it as the verdict.

For a domain, `lookup_domain` returns Helios honeypot observations: `stats.events_24h`, `total_events_30d`, time series, top subdomains and ports, and a `recent_events` sample. `status` is `ok`, `dormant`, or `unknown`. Volume and cadence matter more than any single event; a sharp change in `time_series` is the signal worth surfacing.

## Report

- **Verdict**: one line. What this address is.
- **Confidence**: high, medium, or low, and what drives it. Fresh provider attribution is high. An empty response is low, not clean.
- **Evidence**: the specific fields, with `last_seen` recency where it applies.
- **Recommended action**: block, step up, flag, monitor, or allow. Match it to the confidence.
- **What would change the verdict**: the signal that is missing or stale, and what would settle it.

## Guardrails

Never recommend a block on `network.type` alone. Never on an empty `providers[]`. Never on a benign automation category. Never on `risk_score` alone without naming the signals underneath.

If the evidence is thin, say so. "Not enough signal to act on" is a legitimate verdict and a better answer than a manufactured one.
