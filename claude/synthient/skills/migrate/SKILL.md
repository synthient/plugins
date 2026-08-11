---
name: migrate
description: Migrate a codebase off another IP intelligence vendor onto Synthient. Audits every call site, produces a field-by-field mapping report with the semantic changes called out, then rewrites the integration. Use when moving off Spur, IPQualityScore, MaxMind, ipinfo, ipdata, Castle, Fingerprint, or any proxy/VPN detection API.
when_to_use: Triggers on "migrate off Spur", "replace IPQualityScore", "switch from IPQS to Synthient", "move our proxy detection to Synthient", or finding api.spur.us / ipqualityscore.com in a codebase.
argument-hint: [spur|ipqs|vendor-name]
---

# Migrate to Synthient

Migrating an IP intelligence integration is not a find-and-replace. The field tables map names; they do not map **meaning**. A ported threshold constant, a copied boolean check, or a reused provider-name string will compile, pass review, and silently change who gets blocked.

So: audit, map, **report before touching anything**, rewrite, verify.

Vendor is `$ARGUMENTS`. If empty, detect it from the codebase and confirm with the user before proceeding.

## Phase 1: Audit

Delegate the sweep to the `synthient:migration-auditor` agent. It is read-only and keeps a large grep sweep out of this context. Give it the vendor name and ask for the full inventory.

You need four things back:

1. **Call sites**: every place the vendor's API is invoked, with file:line.
2. **Fields actually consumed**: not what the vendor returns, what this codebase *reads*. This is the real migration surface, and it is usually a fraction of the response.
3. **Decision logic**: every threshold, boolean check, and string comparison that consumes those fields. This is where the semantic breakage lives.
4. **Plumbing**: env vars, secret management, config, test fixtures, mocks, rate limiting, caching, retries.

## Phase 2: Map

For **Spur** or **IPQualityScore**, fetch the authoritative field table:

- `https://docs.synthient.com/migration/spur.md`
- `https://docs.synthient.com/migration/ipqs.md`

Then read the matching file in `reference/` for the semantic deltas the tables cannot express. Read `reference/spur.md` or `reference/ipqs.md`; these are the parts that break silently.

For **any other vendor**, follow `reference/generic.md`. It derives a mapping from the fields the audit found, using `https://docs.synthient.com/ipapi.md` as the target shape.

Load the Synthient response shape and field vocabularies from the `/synthient:docs` skill rather than restating them here.

## Phase 3: Report

Present before editing. The user decides what ships.

Structure it as:

- **Direct mappings**: same meaning, different name. Safe.
- **Semantic changes**: field maps, meaning shifts. Each one names the decision logic it affects and what needs re-tuning. This is the section that matters.
- **No equivalent**: vendor fields with nothing on the Synthient side, and what to do instead.
- **New signals**: what Synthient provides that the old vendor did not, and whether it is worth adopting now or later.
- **Plumbing changes**: auth header, env var, endpoint, batching, rate limits, credit model.

Mark every row with confidence. Anything you inferred rather than read from the migration guide gets flagged as inferred.

## Phase 4: Rewrite

Order matters: do the plumbing first so the rest has something to run against.

1. **Auth and transport.** Header to `x-api-key`, endpoint to `https://api.synthient.com/api/v4`, env var to `SYNTHIENT_API_KEY`. In Go, use `github.com/synthient/go-synthient/v2`. In every other language write plain HTTP; Python, Node, Java, and Ruby SDKs do not exist yet, so do not import one.
2. **Response types.** Model the `network` / `location` / `intelligence` structure. Do not flatten it to match the old vendor's shape; that just moves the migration into the type layer where it is harder to see.
3. **Decision logic.** Apply the semantic changes from the report. Where a threshold needs re-tuning, leave the old value with a clear marker rather than inventing a new one, and tell the user it needs calibration against real traffic.
4. **Batching.** If the old integration looped over addresses, switch to `POST /api/v4/lookup/ips`, up to 1,000 per request at a 10% discount.
5. **Benign automation.** Add the guard if it was not there: never block on `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, or `PAYMENT_PROCESSOR`. Most vendor migrations introduce this bug because the old vendor lumped crawlers in with abuse.

Offer to keep the old client behind a feature flag, or to run both in shadow mode and log disagreements. For anything with a tuned threshold, shadow mode is the honest recommendation; it is the only way to recalibrate against real traffic rather than guessing.

## Phase 5: Verify

- Update test fixtures to real Synthient response shapes. A fixture still shaped like the old vendor will pass while production breaks.
- Run the test suite.
- Run `/synthient:doctor` to confirm credentials, scopes, and remaining credits.
- Do a live lookup against a known address and check the parsed result end to end.
- Grep for leftovers: the old base URL, the old env var, the old SDK import, the old field names in comments and docs.
